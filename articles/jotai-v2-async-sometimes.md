---
title: "Jotai v2を使いこなすために実は必須級な“async sometimes”パターンの解説"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "jotai"]
published: true
---

この記事は[株式会社カオナビ Advent Calendar 2024](https://qiita.com/advent-calendar/2024/kaonavi)の23日目の記事です。

**[Jotai](https://jotai.org)** は、Reactで使えるステート管理ライブラリとしては、現状筆者が最も好んでいるものです。その理由は単純で、ステート管理アーキテクチャとして優れていると思うからです。Recoilが現役のころは同じ理由でRecoilを好んでいました。

Jotaiは2023年1月にv2がリリースされました。非同期処理の扱いがv2はそれより前と異なっており、簡単に言うとJotaiのコアから非同期処理（Promise）に対する特別扱いが排除されました。これにより、コアのAPIをReactから切り離すことができたとされています。JotaiはReactから使われることが多いとはいえ、以下のスライドでもJotaiが「Reactに依存しないライブラリ」として評価されていることからも分かるように、このような特徴は技術の普及に寄与します。

https://speakerdeck.com/takonda/minimize-framework-dependency-in-frontend

しかし、筆者がJotaiを使ってみて気づいたことは、特にJotai v2で非同期処理を扱う際には、独特のテクニックが必要だということです。それが、この記事で紹介する **[async sometimesパターン](https://jotai.org/docs/guides/async#async-sometimes)** です。これはJotaiのドキュメントに記載されている公式（？）用語です。

これは、簡単に言えば、atomの中身を`T | Promise<T>`とし、場合によって同期的に計算されることもあれば非同期的に計算されることもあるようにするというパターンです。

特にReact 19において、[同期的に計算できるステートを非同期的に計算することは不必要なUIの遅延を生む](https://github.com/facebook/react/issues/31819)ため、ステートが非同期に計算されるかもしれない場合でも、同期的に計算できるときはそうすることが重要です。

この記事は、以下のスライドの公開後に有識者の皆さんに教えていただいたり議論して得たりした知見をまとめたものです。

https://speakerdeck.com/uhyo/react-19-plus-jotaiwoshi-siteqi-duita-zhu-yi-dian

## 非同期atomを使う例

筆者がおすすめするjotaiの使い方は、データフェッチング等を含む各種ロジックをコンポーネントから切り離し、jotaiの中で（atomとして）実装することです。そうすると、必然的に非同期atomが増えてきます。

昔のjotai（v1.x）では「非同期atomに依存する同期atom」というのも可能だったのですが、v2では前述のようにコアから非同期処理に対する特別扱いが無くされた結果として、非同期atomに依存するatomは必然的に非同期atomになるようになりました。これは、async関数を呼び出してawaitする関数もまたasync関数でなければならないことと似ています。

ここでは、具体例としてjotai製の簡単なアプリケーションを用意しました。

https://github.com/uhyo/react-19-jotai-sample/tree/step1

まず、`step1`タグをチェックアウトして動作させてみてください（`npm run dev`）。このようなアプリケーションが表示されるはずです。

![アプリケーションのスクリーンショット。右半分に47都道府県に対応するチェックボックスが並んでおり、全てチェック済みである。左半分には次のように表示されている。「47個の都道府県の合計 = 123,855,977 平均 = 2,635,233.553](/images/jotai-v2-async-sometimes/app-screenshot.png)

これは、47都道府県の人口データの合計と平均を表示するアプリケーションです。画面のチェックボックスを操作することで、どの都道府県を合計・平均の計算に含めるかを選択できます。

お察しのとおり、マスターデータ（47都道府県の人口データ）を非同期で取得するという想定になっています。マスターデータは次のようなJSONであり、都道府県の名前もここから取得します。

```json
{
  "北海道": 5051096,
  "青森県": 1163606,
  ...
}
```

この段階（step1）では、標題にあるasync sometimesパターンを使っていません。それゆえに起こる問題として、チェックボックスを操作するたびに画面右半分が0.3秒間ほど「Loading...」になってしまいます。まずは、その原因を紐解きましょう。

前提として、このアプリケーションではSuspenseを使っており、画面左半分と右半分がそれぞれ別のSuspenseの範囲になっています。左半分は、データの計算に0.5秒かかるという設定で作ってあるのでチェックボックスを操作するたびに0.5秒間「Loading...」となり、これは正常です。一方、右半分は、時間のかかる計算が無いはずなのに「Loading...」表示が挟まってしまうのが問題です。

### ステート設計

このアプリケーションのステート設計を見てみましょう。

前述のとおり、データの取得もjotaiに載せています。マスターデータの取得はこうなっています。

```ts:state/data.ts
import { atom } from "jotai";
import { fetchData } from "../data/fetchData";

export const dataAtom = atom(() => fetchData());

export const prefecturesAtom = atom(async (get) => {
  const data = await get(dataAtom);
  return Object.keys(data);
});
```

`dataAtom`はマスターデータを取得するためのatomで、`prefecturesAtom`は都道府県の名前のリストを取得するためのatomです。`prefecturesAtom`は`dataAtom`を参照しているため、`dataAtom`が非同期atomであることから`prefecturesAtom`も非同期atomになっています。

次に、チェックボックスの状態の設計です。複数のチェックボックスの状態を管理したい場合`Set`を使うのが有効なので、そのような設計になっています。ポイントは、初期値が全チェックであり、その状態を表現するためには都道府県の名前のリスト（`prefecturesAtom`）が必要であることです。チェックボックスの状態は以下のコードの`checksAtom`で実装されています（以降、import宣言などは省略します）。

```ts:state/checks.ts （抜粋）
const internalChecksAtom = atom<Set<string> | undefined>(undefined);

export const checksAtom = atom(
  async (get) => {
    const internal = get(internalChecksAtom);
    // 初期状態は全チェック
    if (internal === undefined) {
      return new Set(await get(prefecturesAtom));
    }
    return internal;
  },
  (_get, set, checks: Set<string>) => {
    set(internalChecksAtom, checks);
  },
);
```

今回は、`checksAtom`の内部実装として`internalChecksAtom`を用意して、こちらは同期atomとし、初期状態を`undefined`で表現しています。`checksAtom`は、初期状態が`undefined`の場合は都道府県の名前のリストを取得して`Set`に変換して返し、それ以外の場合は内部状態をそのまま返します。また、`checksAtom`への書き込みはそのまま内部状態に書き込むようにしています。

このように、初期状態が非同期処理に依存する場合に、内部状態と外に見せる状態を分離させるのはjotaiに限らずステート管理で使えるテクニックです。

ここで注目すべき点は、チェックボックスの状態を表す`checksAtom`が非同期atomになっていることです。その理由は、初期状態が非同期処理に依存しているからです。お察しのとおり、この問題が先ほど紹介した0.3秒の「Loading...」表示の原因です。

余談ですが、チェックボックスをクリックしたときに状態を更新する処理もjotaiで実装しています。このような場合、書き込み専用のatomを作ってそこにステート更新ロジックを載せることで、ステート更新処理もjotaiに寄せることができます。

```ts:src/checks.ts （抜粋）
export const toggleCheckAtom = atom(
  null,
  async (get, set, prefecture: string) => {
    const checks = await get(checksAtom);
    const newChecks = new Set(checks);
    if (checks.has(prefecture)) {
      newChecks.delete(prefecture);
    } else {
      newChecks.add(prefecture);
    }
    set(checksAtom, newChecks);
  },
);
```

今回の記事にはあまり関係ないので折りたたんでおきますが、これらのatomを用いるチェックボックス表示のコンポーネントの実装はこのようになっています。

:::details コンポーネントの実装
```tsx:src/ui/Checks.tsx
export const Checks: FC = () => {
  const prefectures = useAtomValue(prefecturesAtom);
  const checks = useAtomValue(checksAtom);
  const toggle = useSetAtom(toggleCheckAtom);

  return (
    <>
      <ul
        css={{
          listStyleType: "none",
          display: "grid",
          gap: "4px",
          gridTemplateColumns: "repeat(auto-fill, minmax(100px, 1fr))",
        }}
      >
        {prefectures.map((prefecture) => (
          <li key={prefecture}>
            <label>
              <input
                type="checkbox"
                checked={checks.has(prefecture)}
                onChange={() => {
                  toggle(prefecture);
                }}
              />
              {prefecture}
            </label>
          </li>
        ))}
      </ul>
    </>
  );
};
```
:::

### 0.3秒のサスペンドの原因

結局、チェックボックスをクリックしたあと0.3秒間「Loading...」表示が出てしまう原因は、`checksAtom`が非同期atomであることにあります。

JotaiはSuspenseに対応したライブラリですから、非同期atomを読み込む際にサスペンドしてくれます。具体的には、`useAtomValue`に非同期atom（Promiseを値とするatom）を与えた場合には、サスペンドして中身を取り出してくれます（React 19での`use`のような挙動です）。

チェックボックスをクリックした際には`checksAtom`の値が変わります（非同期atomなので、新しい結果を含んだPromiseになります）。この新しいPromiseは一瞬で解決します（`await get(prefecturesAtom)`がすでに解決済みのPromiseのawaitとなるため）。しかし、一瞬とはいえPromiseからの読み出しなのでサスペンドが発生します。

サスペンドの原因となったPromiseは一瞬で解決するはずですが、React 19ではこの場合にもサスペンドが明けるまで0.3秒待たされる仕様となっています。これが0.3秒間「Loading...」が表示される原因です。

ちなみに、React 18では0.3秒間待たされないので、一瞬サスペンドして一瞬で再描画される挙動となります。これは速度的には問題なかったのですが、一瞬画面がちらついてしまうのでどちらにせよ良くありません。

## Async sometimesパターンによる解決

では、async sometimesパターンを用いてこの問題を解決してみましょう。いきなり答えになってしまいますが、`checksAtom`をこのように書き換えます。書き換え後ソースコードはリポジトリの`step2-async-sometimes`タグにあります。

```ts:state/checks.ts
export const checksAtom = atom(
  (get) => {
    const internal = get(internalChecksAtom);
    // 初期状態は全チェック
    if (internal === undefined) {
      return get(prefecturesAtom).then((prefectures) => new Set(prefectures));
    }
    return internal;
  },
  (_get, set, checks: Set<string>) => {
    set(internalChecksAtom, checks);
  },
);
```

もともと`checksAtom`は`WritableAtom<Set<Promise>, ...>`型だったのですが（`...`部分は省略）、このように変更することで、`checksAtom`は`WritableAtom<Set<string> | Promise<Set<string>>, ...>`型になります。つまり、中身がPromiseかもしれないしそうではないかもしれないということで、言い換えれば中身が同期的に取得できるかもしれないし、非同期的に取得できるかもしれないということです。

中身を見ると、まず`checksAtom`の中身を計算する関数がasync関数ではなくなっています。これにより、`checksAtom`は同期的に計算できる場合は同期的に計算されるようになります。同期的には計算できない場合、具体的には初期状態の場合のみ、値がPromiseになります。

こうするだけで、もう問題が解決しているはずです。チェックボックスをクリックしても、画面右側は「Loading...」にならずすぐに表示が更新されます。これは、`checksAtom`の中身がPromiseではない場合（同期的に計算できた場合）はコンポーネントがサスペンドしないためです。

要するに、async sometimesパターンは文字通り「非同期的なこともある」という実装を指すものであり、サスペンドの必要が無い場合にサスペンドするのを避けるために有効です。

筆者は正直のところ、React 19に`use` APIが導入されたこともあって、React 19ではPromiseが今までよりも気軽に使えると思っていました。しかし、少なくとも現状では、サスペンドはUIを無駄に遅延させる可能性があるため、データ取得のように本当の非同期処理が必要な場合にはPromiseを避けるべきです。そのことと、jotaiにロジックを寄せることを両立するために、async sometimesパターンが非常に有用だと感じました。

## Async sometimesパターンを助けるユーティリティ: jotai-derive

ところで、このような非同期atomがあったとします。

```ts
export const checkCountAtom = atom(async (get) => {
  const checks = await get(checksAtom);
  return checks.size;
});
```

元々`checksAtom`が非同期atomであるため、`checkCountAtom`も非同期atomになってしまいます。しかし、`checksAtom`がasync sometimesパターンを使っている場合、`checkCountAtom`にも同様に改善の余地があるはずですね。`checksAtom`の中身がPromiseである場合にのみ`checkCountAtom`がPromiseになるようにしたいです。

これは手で実装するとこんな感じになります（任意のthenableに対応するために`'then' in checks`という手もありますが）。

```ts
export const checkCountAtom = atom(
  (get) => {
    const checks = get(checksAtom);
    if (checks instanceof Promise) {
      return checks.then((checks) => checks.size);
    }
    return checks.size;
  },
);
```

しかし、このような実装を手書きするのはつらいので、このパターンのためのユーティリティを備える[jotai-derive](https://jotai.org/docs/third-party/derive)が提供されています。サードパーティライブラリという建付けですがjotaiの公式ドキュメントに同居しています。

まず、jotai-deriveから提供されている`soon`を使うと、実装をこのように書き換えられます。

```ts
import { soon } from "jotai-derive";

export const checkCountAtom = atom((get) => {
  return soon(get(checksAtom), (checks) => checks.size);
});
```

これは前述のような分岐を内部で行い、`get(checksAtom)`の結果がPromiseなら`soon`の返り値もPromiseになるし、そうでない場合は`soon`の返り値も同期的に計算されます。

さらに、より簡潔に書けるようにするために`derive`も提供されています。これを使う場合は次のように書けます。

```ts
import { derive } from "jotai-derive";

export const checkCountAtom = derive([checksAtom], (checks) => checks.size);
```

つまり、依存先のatomを宣言しておくことで、`get`して`soon`を噛ませるところまでやってくれます。

`derive`は「依存先がasync sometimesなatomだけど、自分は同期的に計算できるatom」を使う場合にとても便利です。より複雑なケースでは`soon`が使えるほか、`Promise.all`のsoon版のような`soonAll`も提供されています。

## まとめ

この記事では、筆者が推奨しているように非同期処理などもなるべくjotaiに載せるアーキテクチャを採用している場合に、なるべくサスペンドを避ける必要があることを説明しました。そして、そのためにはasync sometimesパターンが有用であることを示しました。

不必要なサスペンドを避けるという考えはReactを使う上でどうやら重要なようで（筆者としてはReactの方針にあまり納得がいっていないのでissueを建てたりもしていますが）、その状況下jotaiを最大限活かすためには、標題の通りasync sometimesパターンは必須級と言えるでしょう。