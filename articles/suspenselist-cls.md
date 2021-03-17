---
title: "ReactのSuspenseListでお手軽CLS対策"
emoji: "🚸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

みなさん、React の[Concurrent Mode](https://reactjs.org/docs/concurrent-mode-reference.html)使っていますか？　まだという方もまだまだ遅くはありませんのでご安心ください。

この記事では、Concurrent Mode API の一つである`SuspenseList`を使って、[Core Web Vitals](https://web.dev/vitals/)の一つである Cumulative Layout Shift (**CLS**) の発生を抑制する方法を紹介します。

## SuspenseList とは

SuspenseList は React に組み込みのコンポーネントで、複数の`Suspense`コンポーネントを子として持ち、それらが表示される順番を制御する機能を持ちます。Suspense についても一応復習しておくと、これは「内部でサスペンドしたコンポーネントがあった（＝中身がまだ読み込み中である）場合は中身の代わりに指定されたフォールバックコンテンツを表示する」というコンポーネントであり、Concurrent Mode のコアとなるパーツの一つです。

SuspenseList を使うことで、「複数の Suspense コンポーネントの中身は必ず上から順番に表示する」という制御ができます。例えば、上と下の 2 つの Suspense があった場合、下が先に読み込み完了しても上が読み込み完了するまで下は表示されず、下が読み込み完了した時点で両方が表示されます。逆に、上が先に読み込み完了した場合はまず上だけが表示されます。その後、下が読み込み完了したら下も表示されます。

以上の説明だけで全てを理解された方もいるでしょう。そうです、CLS を発生させそうなコンポーネントを上から順番に表示すればいいのです。

これだけで記事を終わりにしてもよいのですが、以下ではもう少し具体的にやり方を説明していきます。

## CLS の原因と対策

CLS が発生する主な要因となるのは、データの読み込みです。ページの初期レンダリング後に API などからデータを読み込む場合、読み込み中の表示から読み込み後の表示に切り替わる際にコンテンツの高さが変わってしまうと CLS が発生します。

CLS に対する最も基本的な対策は、読み込み中も読み込み後もコンテンツの高さを一緒にすることです。しかし、API から読み込んだデータの内容に応じてコンテンツの高さが変わるようなような場合、ローディング中にコンテンツの高さを知ることができず、この方法がとれません。

そもそも読み込み中と読み込み後ので高さが変わるのがまずい理由は、高さが変わることでそれより下のコンテンツの表示位置が画面表示上ずれる（＝ Layout Shift が発生する）こと、そしてそれにより悪いユーザー体験が発生することです。

であれば、読み込み中の領域よりも下に何も表示しなければ Layout Shift が起こりません。アプリのデザインによっては読み込み中にユーザーに見せられるものが少なくなるという問題はありますが、ともあれこの方法ならば読み込み後のコンテンツの高さが分からなくても CLS を回避できます。この記事をご覧の方の中には、このような方法で CLS 対策を実装した覚えがある方もいるのではないでしょうか。

このような方針で CLS を対策する際に大きな助けとなるのが SuspenseList です。

## SuspenseList の利用法

いきなりですが、実際に SuspenseList を使ってこれを実装してみたものをお見せします。次のデモでは、高さの異なる 5 つのボックスがバラバラに読み込まれ、縦に並んで表示されます。use SuspenseList というチェックボックスがあり、これを切り替えることで SuspenseList を使用する場合と使用しない場合の挙動を見ることができます。

SuspenseList を使わずに 5 つのボックスをただ並べた場合、5 つのボックスは読み込まれた順に表示されます。その結果、最初一番上に表示されていたボックスが下に追いやられるのが見られます。また、今回は footer も最初から表示していますが、これも下に追いやられていきます。これが CLS です。

一方で、SuspenseList を使った場合、CLS は発生しません。一度表示されたボックスは動かず、必ず追加のボックスは下に表示されていきます。footer はボックスが全て読み込まれてから表示されます。その一方で、SuspenseList ありの方がやや表示に時間がかかる印象を受けたかもしれません。これは、上のボックスの読み込みが遅い場合にそれより下の表示が全てブロックされてしまい、何も表示されない時間が期待的に長くなるからです。

@[codesandbox](https://codesandbox.io/embed/suspenselist-cls-pdmlm?fontsize=14&hidenavigation=1&theme=dark)

上のデモのコードから SuspenseList 使用部分を抜き出すと、こうなっています。

```tsx
const DataList: React.VFC<{
  data: LoadedData[];
  useSuspenseList: boolean;
}> = ({ data, useSuspenseList }) => {
  if (useSuspenseList) {
    return (
      <SuspenseList revealOrder="forwards">
        {data
          .map((data, index) => {
            return (
              <Suspense fallback={null} key={index}>
                <OneData data={data} num={index + 1} />
              </Suspense>
            );
          })
          .concat([
            <Suspense fallback={null} key={-1}>
              <Footer />
            </Suspense>,
          ])}
      </SuspenseList>
    );
  } else {
    return data
      .map((data, index) => {
        return (
          <Suspense fallback={null} key={index}>
            <OneData data={data} num={index + 1} />
          </Suspense>
        );
      })
      .concat([
        <Suspense fallback={null} key={-1}>
          <Footer />
        </Suspense>,
      ]);
  }
};
```

if 文で分岐していますが、どちらのコードも`SuspenseList`で囲まれているかどうかの違いを除いて全く同じです。`OneData`コンポーネントが一つのボックスの読み込みと表示を担当するコンポーネントで、読み込み中はサスペンドします。

`SuspenseList`の使い方としては、このように`Suspense`たちを囲むように使うことで効果を発揮します。ちなみに、`revealOrder`という prop がありますが、これは前から順番に表示する`forwards`、後ろから表示する`backwards`、全部読み込みが完了したら一気に表示する`together`が今のところ選択可能です。今回`Footer`はサスペンドするコンポーネントではありませんが、このように他と同じように`Suspense`で囲むことで`SuspenseList`の制御に組み入れることができます。

## まとめ

この記事では、`SuspenseList`を用いて CLS の発生を回避する方法を説明しました。Concurrent Mode 時代の React では非同期なデータの読み込みを従来よりもさらに宣言的な方法で書けるようになり、React が組み込みでそれを理解することで高いレベルのユーザー体験をよい設計で提供することができます。今回は Concurrent Mode の一部として提供される予定の`SuspenseList`の活用例として CLS 対策に用いる方法を紹介しました。
