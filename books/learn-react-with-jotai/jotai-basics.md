---
title: "jotaiの基本"
---

この章では、Suspenseの話題に入る前に、**jotai**の基本的な使い方を解説します。jotaiは非常にシンプルな状態管理ライブラリであり、基本的な概念を理解するのに多くの時間はかかりません。この章を通じて、jotaiの主要な機能とその使い方を学び、以降の章でSuspenseと組み合わせて使用する準備を整えましょう。

## jotaiの公式サイト

https://jotai.org/

ライブラリの使い方を学ぶには公式サイト・公式ドキュメントが一番なのは間違いありません。

jotaiのドキュメントにも有用な情報は多く載っていますが、個人的には少し分かりにくいと感じています。明確なAPIリファレンスが無く、APIの説明やユースケースの説明、コンセプトの説明などが入り混じっているからではないかと思います。

この章では、ユースケースをベースにAPIを紹介し、jotaiの概念を学びます。

## `useAtom`を`useState`のように使う

Reactの`useState`はステートに関する最も基本的なAPIであり、それに近いjotaiのAPIが`useAtom`です。`useAtom`の使い方は次のとおりです。

```ts
// コンポーネントの外側のどこかで
const countAtom = atom(0);

// コンポーネントの中で
const [count, setCount] = useAtom(countAtom);
```

このコードでは、`atom`を使って`countAtom`を定義しておいて、`useAtom`で`countAtom`を使用しています。

ポイントは、**`useState`が持っていた2つの役割が`atom`と`useAtom`に分割されている**ことです。すなわち、「ステートの定義」と「ステートの読み書き」です。

| | React | jotai|
| - | - | - |
| ステートの定義 | `useState` | `atom` |
| ステートの読み書き | `useState` | `useAtom` |

この分割により、jotaiではReact本体の機能よりも柔軟なステート管理アーキテクチャが実現します。

`countAtom`のようなオブジェクトは、名前から分かるとおり**atom**と呼ばれます。上述のように、atomはステートの定義を表すオブジェクトです。

ステート管理ライブラリらしい特徴として、このように作られたatom（上の例でいう`countAtom`）は**グローバルな状態**となります。つまり、複数個所（同じコンポーネントの複数使用でも、別々のコンポーネントでも）で`countAtom`を使うことができ、その場合はそれらは同じステートを共有します。`countAtom`の中身の更新は全ての`countAtom`使用箇所に同時に反映されるし、どこからでも同じ`countAtom`に書き込むことができます[^note_provider]。

[^note_provider]: 細かいことを言えば、[Provider](https://jotai.org/docs/core/provider)を使えばステートの保管場所（Store）を複数持つこともできますが、この本の主題からは外れてしまうので解説しません。

### 読み取りのみ・書き込みのみの使用

`useAtomValue`と`useSetAtom`は、`useAtom`の機能（読み書き）のうちどちらか一方のみ使いたい場合に有効です。

```ts
// どちらも
const [count, setCount] = useAtom(countAtom);
// 読み取りのみ
const count = useAtomValue(countAtom);
// 書き込みのみ
const setCount = useSetAtom(countAtom);
```

特に`useSetAtom`は、自身は`countAtom`の値を使用しないので、`countAtom`の値が更新されても再レンダリングが起こらず、パフォーマンス的に若干有利になるという利点があります。

## 派生atom

atomには、プリミティブatomと派生atomの2種類が存在します。先ほどの例の`countAtom`はプリミティブatomです。

プリミティブatomと派生atomの共通点（全てのatomが備える特徴）は、`useAtom`などのフックを用いて**読み書きができる**ことです（ただし、派生atomの場合は読み取り専用・書き込み専用の場合もあります）。

プリミティブatomと派生atomの違いは、**自身がステートを保持するかどうか**です。プリミティブatomはステートを保持しますが、派生atomは保持しません。その代わり、派生atomに読み書きしようとした場合には関数が実行されます。

以下の例では、`countDisplayAtom`を派生atomとして定義しています。例から分かるように、派生atomも同じ`atom`関数を使って定義します。

```ts
// プリミティブatom
const countAtom = atom(0);
// 派生atom
const countDisplayAtom = atom((get) => {
  const count = get(countAtom);
  return count.toLocaleString();
});

// 使用例（コンポーネント内）
const countDisplay = useAtomValue(countDisplayAtom);
const setCount = useSetAtom(countAtom);
```

派生atomを定義するときは、このように`atom`に関数を渡します。その関数は、例のように`get`を引数で受け取ります。`get`は他のatomの値を取得できる関数です。

この場合、`countDisplayAtom`の値を読み取ろうとした場合、その関数が実行されて結果が計算されます。この例では、`countDisplayAtom`の値は「`countAtom`の値に対して`toLocalString()`を実行した値」になります。例えば`countAtom`の値が`0`であれば`countDisplayAtom`の値は`"0"`になるし、`countAtom`の値が`1000`であれば`countDisplayAtom`の値は`"1,000"`になる……といった具合です（実行環境によります）。

jotaiは`get`の呼び出しを通じてatom間の依存関係をトラッキングしています。これにより、「`countDisplayAtom`は`countAtom`に依存しているので、`countAtom`の値が更新された場合は`countDisplayAtom`の値を再計算する。その結果、`countDisplayAtom`を読んでいるコンポーネントも再レンダリングする」といった動作が可能になっています。

この`countDisplayAtom`は、読み取り時の関数しか与えていないので、読み取り専用atomになります。

### 派生atomに書き込む

jotaiの面白い文化として、「派生atomへの書き込み」や、何なら「書き込み専用atom」まで利用する点があります。上記の例では読み取り用の関数しか与えていませんでしたが、書き込み用の関数を与えることで書き込み可能な派生atomとなります。

`atom`関数を使って派生atomを作る方法は、まとめるとこうです。

```ts
const 派生atom = atom(読み取り関数, 書き込み関数);
```

2つの関数を両方与えると読み書き可能な派生atomとなります。次の例では、読み取り関数は与えない（`null`を渡す）ことで書き込み専用atomを表現しています[^note_write_only]。

[^note_write_only]: この場合は読み込むと常に`null`が返るatomとなるので、正確には「書き込み専用」というよりは「読み込んでも意味がない」派生atomと言えるでしょう。

例えば、次のような書き込み専用atomが考えられます。

```ts
const countAtom = atom(0);

const incrementAtom = atom(
  null,
  (get, set) => {
    const currentCount = get(countAtom);
    set(countAtom, currentCount + 1);
  },
);

// 使い方
const increment = useSetAtom(incrementAtom);

// countAtomの値に1を足す
increment();
```

派生atomの書き込み用の関数は、このように`get`と`set`を受け取ります。この関数は、その派生atomに書き込もうとしたときに呼び出されます。この関数の中で、`get`と`set`を駆使して必要なステート操作を行うのです。

この例の`incrementAtom`は、書き込まれたら`countAtom`に1を足すという挙動をします。

### 派生atomの書き込み時の引数

書き込みができる派生atomの面白いところは、**書き込み時の引数も柔軟に定義できる**ことです。上の例の`incrementAtom`は、書き込み（`increment()`）時に引数を要求していませんね。

一方でプリミティブatom（`countAtom`）にも書き込み可能ですが、あちらは`setCount(100)`のように引数を要求します。

このように、書き込み可能なatomは、定義次第で引数の受け取り方を変えることができるのです。

例えば、`incrementAtom`の定義を変えて、書き込み時に引数を渡せるようにするのは、こうします。

```ts
const incrementAtom = atom(
  null,
  (get, set, step = 1) => {
    if (step < 0) {
      throw new Error("負の数を足すことはできませんよ！！");
    }
    const currentCount = get(countAtom);
    set(countAtom, currentCount + step);
  },
);

// 使い方
const increment = useSetAtom(incrementAtom);

// countAtomの値に1を足す
increment();
// countAtomの値に100を足す
increment(100);
```

このように、書き込み時の関数を定義する際に`get`、`set`に続けて引数を受け取るようにすることで、書き込み時に引数を要求できます。

### 派生atomとカプセル化

jotaiは派生atomを多用する文化です。その理由のひとつに、**カプセル化**と相性がいいことが挙げられます。

例えば、次の内容の`counter.ts`モジュールを作ることを考えます。

```ts
const countAtom = atom(0);

export const countDisplayAtom = atom((get) => {
  const count = get(countAtom);
  return count.toLocaleString();
});

export const incrementAtom = atom(
  null,
  (get, set, step = 1) => {
    if (step < 0) {
      throw new Error("負の数を足すことはできませんよ！！");
    }
    const currentCount = get(countAtom);
    set(countAtom, currentCount + step);
  },
);
```

この例では、`countAtom`はexportされておらず、モジュールの中に隠されています。代わりに`countDisplayAtom`と`incrementAtom`がexportされており、ユーザーはこれらを使って間接的に`countAtom`を読み書きできます。

これによって、以下の効果があります。

- ユーザーは`countAtom`の値として`"1,234"`のような文字列の形式でしか取得できない（フォーマットし忘れやフォーマットの仕様揺れを防げるかも）
- ユーザーは`countAtom`の値を増やすことはできても減らすことはできない

これはまさしくカプセル化ですね。データを保管するプリミティブatomを外に見せないことで自由な操作を防ぎ、派生atomのみを提供することでどんな操作が可能なのかを制御しています。

このように、派生atomを多用して状態を組み立てることで、カプセル化の考え方を採り入れながらステート設計ができるのがjotaiの面白いところです。

## ユーティリティ関数の文化

jotaiには、「atomを作るユーティリティ関数」の文化もあります。これまではjotaiが提供する`atom`関数を使ってatomを作ってきましたが、派生atomを駆使することで、「特殊な挙動をするatom」を作る（ように見える）関数を定義することもできます。

そのようなユーティリティ関数は我々で実装することもできますし、一部はjotaiやその周辺ライブラリから提供されています。例えば、次のページでは「リセット可能」なatomを定義できる`atomWithReset`が紹介されています。

https://jotai.org/docs/utilities/resettable

これは、通常のプリミティブatomのように読み書きできるが、特殊な値 `RESET` を書き込むことでatomの初期値に戻るという追加の挙動を有するatomを作るユーティリティ関数です。

```ts
const countAtom = atomWithReset(0);

// 使用例
const setCount = useSetAtom(countAtom);

setCount(1); // 1になる
setCount(c => c + 1); // 2になる
setCount(RESET); // 0になる
```

使う側で`countAtom`の初期値を知らなくても、リセットすることができます。これもある種のカプセル化、責務の分離と言えます。

## まとめ

この章では、本の内容に入る事前準備として、jotaiの基本的な使い方を学習しました。

また、jotaiの文化として、派生atomやユーティリティ関数を多用したカプセル化の文化があることを学びました。

## 練習問題

ユーティリティ関数の練習として、`atomWithReset`を自分で実装してみましょう。

```ts
const RESET = Symbol();

function atomWithReset<T>(initialValue: T) {
  // どう実装する？
}

// 使用例
const countAtom = atomWithReset(0);

const [count, setCount] = useAtom(countAtom);

setCount(5); // 5になる
setCount(c => c + 1); // +1
setCount(RESET); // 0 に戻る
```

:::details 答え

```ts
function atomWithReset<T>(initialValue: T) {
  const dataAtom = atom(initialValue);
  const resettableAtom = atom(
    (get) => get(dataAtom),
    (get, set, updater: T | ((prev: T) => T) | typeof RESET) => {
      if (updater === RESET) {
        // リセットする
        set(dataAtom, initialValue);
        return;
      }
      set(dataAtom, updater);
    }
  );
  return resettableAtom;
}
```

成果物は1つのatom（`resettableAtom`）ですが、それに付随するものとして`dataAtom`を裏で作っているのがポイントです。

`dataAtom`がデータの保管を担当する“生のatom”であり、それを直接外に見せる代わりに`resettableAtom`を用意してそれを返しています。

`resettableAtom`は書き込み時に呼び出される関数であり、渡された値が`RESET`だったら`initialValue`を書き込む処理を行っています。それ以外の場合はそのまま`dataAtom`に書き込みます。

プリミティブatomは`5`とか`c => c + 1`といった値の書き込みに対応しているので、そのまま`set`してあげればOKです。

このように、atomを作るユーティリティ関数では、外に見える1つのatomの裏に別のatomが備わっていることがよくあります。

:::