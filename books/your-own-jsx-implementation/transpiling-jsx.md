---
title: "JSXのトランスパイル"
---

皆さんこんにちは。このZenn本は、[N代アドベントカレンダー2024](https://adventar.org/calendars/10235)の3日目の記事（？）です。この本では、タイトルにあるように、TypeScriptで自前のJSX実装を作るための知識を紹介します。

**JSX**はJavaScriptに対する拡張構文であり、主にReactなどのUIライブラリで使われています。

```jsx:JSXの例
const element = <h1>Hello, world!</h1>;
```

しかし、JSXはUIライブラリ以外の活用法もあります。JSXは単なる構文であり、それに異なる意味を与えてしまえば、その通りにJSXは動きます。

つい先日も、そのようにJSXを活用してみたという内容のトークがありました。

@[speakerdeck](31c365fffa114d58b4f9c5ca3794d4c5)

そこで、この本では自前のJSX実装を作るために必要な知識を紹介します。特に、TypeScriptの型定義や設定周りも含めて手取り足取り解説します。

## JSXのトランスパイル

JSXが動くのは、JSXをただのJavaScriptコードに**トランスパイル**しているからです。最近は、TypeScriptからJavaScriptにトランスパイルするツールがこれも一緒にやってくれます。

TypeScriptコンパイラ（tsc）もJSXのトランスパイル機能を持っています。ということで、JSXのトランスパイル結果を見てみましょう。

TypeScriptを以下のように設定します。

```json:tsconfig.json
{
  "compilerOptions": {
    "target": "esnext",
    "jsx": "react-jsx",
    "module": "esnext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "rootDir": "./",
    "outDir":  "./dist"
  },
  "include": ["src"]
}
```

重要なのは`jsx`オプションです。ここでは`react-jsx`を指定しており、これでJSXの変換が有効になります。`react-jsx`となっているのはReactが決めた変換方法だからですが、React以外の用途にもこれを使えます。

:::message
`jsx: "react"`という指定もできますが、これは古い変換方法なのでこの記事では取り扱いません。
:::

では、次のコードをトランスパイルしてみましょう。

```tsx:src/index.tsx
const element = <h1 id="hello">Hello, world!</h1>;
```

トランスパイル結果は次のようになります。

```js:dist/index.js
import { jsx as _jsx } from "react/jsx-runtime";
const element = _jsx("h1", { id: "hello", children: "Hello, world!" });
```

`tsc`を実行すると型エラーになりますが、それは後で見ることにしましょう。TypeScriptは型エラーが出た場合でもトランスパイルできるのが特徴です。

トランスパイル結果を見ると、いくつかのポイントがあります。

- JSXは関数呼び出しに変換されます。呼び出されるのは`react/jsx-runtime`からインポートされた`jsx`関数です（`_jsx`という名前に変更されています）。
- 第1引数はタグ名です。
- いわゆるpropsは第2引数にオブジェクトとして渡されます。`children`プロパティにはタグの中身が入ります。

ネストしたJSXもトランスパイルしてみましょう。

```tsx:src/index.tsx
const element = (
  <div key="aaa">
    <h1 id="hello">Hello, world!</h1>
  </div>
);
```

トランスパイル結果は次のようになります。

```js:dist/index.js
import { jsx as _jsx } from "react/jsx-runtime";
const element = (_jsx("div", { children: _jsx("h1", { id: "hello", children: "Hello, world!" }) }, "aaa"));
```

ネストは、`children`に`_jsx`の返り値が入ることで表現されます。また、`key`プロパティは`_jsx`の第3引数に渡されることが分かります。

## `jsxs`関数

`react/jsx-runtime`には`jsxs`関数もあります。これは、`children`が配列になる場合に使われます。

```tsx:src/index.tsx
const element = (
  <div>
    <h1 id="hello">Hello, world!</h1>
    <p>Goodbye, world!</p>
  </div>
);
```

トランスパイル結果は次のようになります（分かりやすいようにフォーマットしています。以降も同様）。

```js:dist/index.js
import { jsx as _jsx, jsxs as _jsxs } from "react/jsx-runtime";
const element = _jsxs("div", {
  children: [
    _jsx("h1", { id: "hello", children: "Hello, world!" }),
    _jsx("p", { children: "Goodbye, world!" }),
  ],
});
```

`jsxs`関数は、JSX構文において静的に複数の要素がchildrenに入る場合に使われます。次のように元のコードを変えると`jsx`に変わります。

```tsx:src/index.tsx
const children = [<h1 id="hello">Hello, world!</h1>, <p>Goodbye, world!</p>];

const element = <div>{children}</div>;
```

この場合のトランスパイル結果は次のようになります。

```js:dist/index.js
import { jsx as _jsx } from "react/jsx-runtime";
const children = [
  _jsx("h1", { id: "hello", children: "Hello, world!" }),
  _jsx("p", { children: "Goodbye, world!" }),
];
const element = _jsx("div", { children: children });
```

`div`の`children`に配列が入るのは変わりませんが、今回は`jsxs`関数は使われず、`jsx`になっています。

このような`jsx`と`jsxs`の使い分けは、Reactにとっては意味があります。Reactでは配列をレンダリングするときには`key`を指定する必要がありますが、配列とかを使わずにただ要素を並べただけのときは`key`を指定する必要はありません。この2つはJSにトランスパイルした後はどちらも`children`が配列になってしまってそれだけだと区別がつかないので、`jsx`と`jsxs`を使い分けることで違いが表現されています。

皆さんのJSXのユースケースでこの区別が必要かどうかは場合によりますが、いずれにせよ`jsx`と`jsxs`の両方に対応する必要があります。
