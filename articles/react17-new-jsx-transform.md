---
title: "React17におけるJSXの新しい変換を理解する"
emoji: "🔨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "javascript", "typescript"]
published: true
---

[今日発表された公式ブログの記事](https://reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)によれば、React17では**新しいJSXの変換**がサポートされます。これはどういうことなのか、我々にどういう影響があるのかをまとめました。

## JSXの変換とは

ほとんどの人は、Reactを使う際に以下のようなJSX記法を使っているはずです。具体的には次のようなもので、`<div>`のようなHTMLに近い記法がJSXです。

```tsx
const Foo = () => {
  return <div>
    <p id="a">I am foo</p>
    <p key="b">I am foo2</p>>
  </div>;
}
```

これらは純粋なJavaScriptではないため、そのままでは実行できません。そのため、何らかの方法でただのJavaScriptに変換する必要があります。現代では、それを担うのはBabelやTypeScriptです。これらによって、上記のJSXを含むコードは次のように変換されます（下記はTypeScriptによる変換の場合）。

```ts
const Foo = () => {
  return React.createElement("div", null,
    React.createElement("p", { id: "a" }, "I am foo"),
    React.createElement("p", { key: "b" }, "I am foo2"));
};
```

React 17では、**この変換結果が変わります**（より正確には、これまでの方式もサポートしつつ新しい方式のサポートが加わります）。

## 新しいJSXの変換結果

React 17では以下のような変換結果となります（TypeScript 4.1 betaで対応しているはずですが、なぜかうまく動かなかったのでこれは手書きです）。

```ts
import { jsx as _jsx, jsxs as _jsxs } from "react/jsx-runtime";

const Foo = () => {
  return _jsxs("div", {
    children: [
      _jsx("p", { id: "a", children: "I am foo" }, void 0),
      _jsx("p", { children: "I am foo2" }, "b"),
    ]
  }, void 0);
};
```

まず最初に注意すべきことは、あくまでJSXからの変換結果が変わっただけであり、我々が慣れ親しんでいるJSXの書き方自体は何も変わっていないということです（`key`などに絡んでエッジケースがあったりしますが）。ですから、この話はどちらかというとReactの内部実装の変化の話であるということです（JSXを変換する必要があるのでBabelやTypeScriptといった周辺のエコシステムが付き合わされていますが）。

ちなみに、この新しい変換自体は2019年前半から構想されていたもので（[RFCがあります](https://github.com/reactjs/rfcs/blob/createlement-rfc/text/0000-create-element-changes.md)）、足掛け1年以上かけてようやく実現まで漕ぎ着けたことになりますね。

その上で変化した点を見ていくと、いくつか挙げられます。

### `jsx`, `jsxs`が`react/jsx-runtime`からインポートされる

従来はJSXのタグは`React.createElement("div", ...)`のような関数呼び出しに変換されていましたが、これが`_jsxs("div", ...)`のような関数呼び出しに変換されました。関数名が変わったのはさして重要ではないのですが、この**jsxやjsxsのインポートがトランスパイラによって自動的に追加される**という点が重要です。

従来、JSX記法を使用するファイルでは以下の記述を含める必要がありました。

```js
import React from "react";
```

これは、JSXが`React.createElement`に変換されるのであらかじめこちら側で`React`を用意しておく必要があったからです。新しい記法では、`React`をあらかじめインポートしておく必要がありません。

利用者的にはこれは嬉しい変更ですが、React側の思惑としてはこれは副作用でしょう。というのも、古い記法でも`React`を自動的にインポートするのはできないわけでもないという点と、また前述のRFCに以下のような記述があるからです。

> Ideally the element creation should be part of the transpiler's own runtime.

つまり、理想的な形としては`jsx`, `jsxs`といったものすらReact側で提供するのではなく、トランスパイラ側で勝手にやっておいてもらいたいという考えがあるのです。とはいえ、さすがにそれは遠い道のりです。そこで、今回の変更では将来を見据えて`jsx`, `jsxs`といった関数を`react/jsx-runtime`という別のサブパッケージに切り出してより疎結合的にしたのだと考えられます。

### childrenの渡し方の変更

従来の変換結果と新しい変換結果をもう一度見比べてみましょう。

```ts
// 旧
const Foo = () => {
  return React.createElement("div", null,
    React.createElement("p", { id: "a" }, "I am foo"),
    React.createElement("p", { key: "b" }, "I am foo2"));
};

// 新
const Foo = () => {
  return _jsxs("div", {
    children: [
      _jsx("p", { id: "a", children: "I am foo" }, void 0),
      _jsx("p", { children: "I am foo2" }, "b"),
    ]
  }, void 0);
};
```

従来の変換結果では、`div`や`p`の子要素の情報は`React.createElement`の第3引数以降に順番に渡されています。一方、新しい変換結果では第2引数のオブジェクト（これはpropsを表しています）の`children`プロパティとして渡されています。

つまり、従来の変換では`children`が特別扱いされていたのに対して、新しい変換ではpropsの一部として扱われているのです。もともとpropsを受け取るコンポーネント側では子要素は`props.children`でしたから、余計な齟齬が無くなったことになります。Reactの内部実装には詳しくありませんが、実装の簡潔化などに貢献しているのでしょう。

ところで、よく見ると関数が`_jsx`と`_jsxs`の2種類あることに気づいたでしょう。これは、子要素が複数ある場合に`_jsxs`となり、この場合`children`に渡されるのは配列となります。この区別が必要になるのは、ReactがもともとJSX内で配列の使用をサポートしていたからです。例えばこういうものですね。

```tsx
const List: React.FC<{ labels: string[] }> = ({
  labels
}) => {
  return <ul>{
    labels.map(label => <li key={label}>{label}</li>)
  }</ul>;
}

// トランスパイル後
const List = ({ labels }) => {
  return _jsx("ul", {
    children: labels.map(label =>
      _jsx("li", { children: label }, label)
    )
  }, void 0)
}
```

この場合、`_jsx`に`children`として配列が渡されることになります。

Reactで配列をレンダリングしようとする場合、このように配列内の要素に`key`を持たせる必要があります。それは、配列は中身が増減したり場所が入れ替わったりする可能性があるので、React側でそれをトラックできるようにするためです。重要なのは、ここで`children`に渡されているのが**ランタイムに作られた配列**であるということです。

ところが、先ほどの`_jsxs`に渡された配列はどうでしょうか。

```tsx
// 変換前
const Foo = () => {
  return <div>
    <p id="a">I am foo</p>
    <p key="b">I am foo2</p>>
  </div>;
}

// 変換後
const Foo = () => {
  return _jsxs("div", {
    children: [
      _jsx("p", { id: "a", children: "I am foo" }, void 0),
      _jsx("p", { children: "I am foo2" }, "b"),
    ]
  }, void 0);
};
```

こちらは、`children`に配列が渡されてはいるものの、その由来が異なります。この配列は`div`の中に複数の要素が並んでいることを表現するための配列であり、**トランスパイル時に作られた配列**です。トランスパイル時に作られた配列は常に一定の並びであり、毎回要素が増減したりすることがありません。それゆえ、`key`を全ての子要素に付与するといった特別な取り扱いが必要ありません。

まとめると、`children`に渡された配列が**ランタイムに作られた配列**なのか**トランスパイル時に作られた配列**なのかを区別するために`_jsx`と`_jsxs`という2種類の関数が存在しているのです。古い変換ではトランスパイル時に複数の子要素があった場合は「配列が渡される」のではなく「複数の引数が渡される」という変換になったので、ここは区別できていました。

### `key`の取り扱いの変化

再び最初の例を再掲しますが、よく見ると`key`の取り扱い方が変わっています。

```tsx
// 変換前
const Foo = () => {
  return <div>
    <p id="a">I am foo</p>
    <p key="b">I am foo2</p>>
  </div>;
}

// 旧変換結果
const Foo = () => {
  return React.createElement("div", null,
    React.createElement("p", { id: "a" }, "I am foo"),
    React.createElement("p", { key: "b" }, "I am foo2"));
};

// 新変換結果
const Foo = () => {
  return _jsxs("div", {
    children: [
      _jsx("p", { id: "a", children: "I am foo" }, void 0),
      _jsx("p", { children: "I am foo2" }, "b"),
    ]
  }, void 0);
};
```

旧変換では、`key`は他のpropsと同じように第2引数のオブジェクトに入れられていました。一方、新しい変換結果では`key`はpropsの中に入らず、`_jsx`の第3引数に渡される形となっています。その理由は前述のRFCで以下のように述べられています。

> Currently, key is passed as part of props but we'll want to special case it in the future so we need to pass it as a separate argument.

つまり、`key`に対して普通のpropsとは異なる特別な取り扱いをしたい場面がこれから発生するので、それに備えて別に扱うように変更するとされています。この特別な取り扱いが何なのか、もうあるのかこれからできるのかは残念ながら未調査です。

### developmentビルドとproductionビルド

実は、新しい変換ではdevelopmentビルド用とproductionビルド用の変換があります。これまで見てきたのはproductionビルド用の変換です。developmentビルド用の変換では、次のように`jsxDEV`関数（および`jsxsDEV`関数が）代わりに使われ、インポート元も異なります。

```ts
import { jsxDEV as _jsxDEV } from "react/jsx-dev-runtime";
const _jsxFileName = "/path/to/src/react.tsx";

const Foo = () => {
    return _jsxDEV("div", {
      children: "foo"
    }, void 0, false, { fileName: _jsxFileName, lineNumber: 4, columnNumber: 9 }, this);
};
```

注目に値するのは、developmentビルドでは変換時にデバッグ用の付加情報が加わるという点です。具体的には、上の例を見て分かるように、JSXが書かれたファイル名とファイル内の位置です。これにより、開発時にReactが位置情報を含めたワーニングを出してくれることが期待でき、エラーやワーニングの調査の助けになります。

また、最後の引数にさりげなく`this`が追加されていますね。これはクラスコンポーネントの場合に活用されるデバッグ情報で、具体的にはstring ref（クラスコンポーネントで使用できる`ref="foo"`のような古い書き方）に対してワーニングメッセージを出す目的で使われます。関数コンポーネントの場合は特に意味はありません。

以上がJSXの新しい変換の概説でした。

## 我々は何をすればいいのか

これまで述べてきた通り、この話はReactの内部実装の話です。また、React 17以降も旧方式はサポートされ続けるので、何もしないという選択肢もあります。

しかし、新しいJSXの変換を採用することによって、より情報量の多いエラーメッセージや、`React`を自分で書く必要が無くなるといった恩恵を受けることができます。また、公式ブログによれば新しい変換の方が少しだけコードサイズ（バンドルサイズ）が減るようです。

まず第一に、**React 17を待つ**ことです。今のうちに試してみたい方は、React 17 RCが出ているので試してみましょう。ちなみに、最初から「React 17で新しい方式がサポートされる」と述べていましたが、公式ブログによると、これはReact 17の正式リリース後に16以前にもバックポートされるようです。つまり、React 16以前でも新しいJSX変換方式が使えるようになる見込みであるということです。何にせよ、対応バージョンが出たら**Reactのバージョンを上げる**ことが必要になります。

次に、実際にJSXからJavaScriptへの変換をするのはReactではなくトランスパイラですから、**トランスパイラのバージョンを上げて設定を変える**ことが必要になります。公式ブログによれば、Babelは7.9.0から、TypeScriptは4.1.0から新方式に対応します。また、Next.jsやGatsbyなどもバージョンを上げると自動的に新方式を使う設定になっているようです。

BabelやTypeScriptを使っている場合は、これまでと異なるポイントとして**developmentビルドとproductionビルドを使い分ける**設定が必要になります。筆者はTypeScript使いなのでTypeScriptの場合について詳説します。

### TypeScriptの設定

これまで、TypeScriptで（React向けに）JSXを使う場合は`jsx`コンパイラオプションを`"react"`に設定していました。tsconfig.jsonの場合は次のような感じですね。

```json
{
  "compilerOptions": {
    // ...
    "jsx": "react",
    // ...
  }
}
```

新方式で変換するには、developmentビルドの場合`"jsx": "react-jsxdev"`とし、productionビルドの場合は`"jsx": "react-jsx"`とします。両者を使い分けるために、`tsc`を素で使っている場合は`tsconfig.json`を2つ用意したり、`ts-loader`の場合は`compilerOptions`で`"jsx"`の値を場合によって指定したりしましょう。

`"jsx"`コンパイラオプションに設定可能な値を、従来からあるものも含めてまとめると次のようになります。今回加わるのは最後の2つですね。

| 値 | `<div />`の結果 | 説明 |
| -- | -- | -- |
| `preverse` | `<div />` | 変換しない。他のツールでJSXを変換するとき向け |
| `react-native` | `<div />` | 変換しないが、拡張子を`.jsx`ではなく`.js`で出力する |
| `react` | `React.createElement("div", ...)` | 旧変換 |
| `react-jsx` | `jsx("div", ...)` | 新変換（productionビルド用） |
| `react-jsxdev` | `jsxDEV("div", ...)` | 新変換（developmentビルド用） |

また、前述の通りJSX記法を使用するためだけに`import React from "react"`を書いていた場合は必要なくなります。消しましょう。Reactチームは、丁寧にも必要なくなったインポートを自分で消すツールを提供してくれています。詳しくは公式ブログをみていただきたいですが、以下のコマンドで実行することができます。

```sh
npx react-codemod update-react-imports
```

## まとめ

この記事では、React 17のリリースと同時に使えるようになるJSXの新しい変換について解説しました。新しい変換によって開発時のエラーメッセージの改善や多少のコードサイズの削減などが期待できますから、積極的に受け入れましょう。