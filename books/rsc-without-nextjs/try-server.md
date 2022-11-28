---
title: "とりあえずサーバー用パッケージを動かしてみる"
---

まず手始めに、とりあえずReact Server Componentsを動かすのに必要なパッケージをインストールして使ってみましょう。この章の開始時の内容に対応するコードはリポジトリの`step1`ブランチにあります。

`package.json`の内容を抜粋するとこんな感じです。React関連のパッケージは、RSCがまだ正式リリースされていないので`experimental`なものをインストールする必要があります。

```ts
  "devDependencies": {
    "@types/react": "^18.0.25",
    "@types/react-dom": "^18.0.9",
    "prettier": "^2.8.0",
    "react": "^0.0.0-experimental-2655c9354-20221121",
    "react-dom": "^0.0.0-experimental-2655c9354-20221121",
    "react-server-dom-webpack": "^0.0.0-experimental-2655c9354-20221121",
    "typescript": "^4.9.3",
    "webpack": "^5.75.0"
  }
```

明らかに浮いているのが`react-server-dom-webpack`ですね。何か名前に`webpack`と入っていて浮いていますが、今は気にしません（RSCの実用上モジュールごとにServer ComponentになったりClient Componentになったりするため、バンドラ側の協力が必要となっています）。`webpack`も入っていますがこれはpeer dependencyの都合です。

## コンポーネントを用意する

まず適当なコンポーネントを用意します。

```tsx:src/app/App.tsx
export const App: React.FC = () => {
  return (
    <div>
      <h1>React Server Components example</h1>
      <p>Hello, world!</p>
    </div>
  );
};
```

## コンポーネントをレンダリングしてみる

サーバー（Node.js）向けのReactレンダリングAPIとしては`renderToPipeableStream`が`react-dom/server`から提供されているのが知られていますが、今回はRSC対応のために、`react-server-dom-webpack`から提供されているものを使います。

具体的なコードはこうです（1行目の`@ts-expect-error`は`react-server-dom-webpack`の型定義が存在していないため必要です）。

```tsx:src/index.tsx
// @ts-expect-error
import rsdws from "react-server-dom-webpack/server";
const { renderToPipeableStream } = rsdws;

import { App } from "./app/App.js";

renderToPipeableStream(<App />).pipe(process.stdout);
```

これをJSにコンパイルしてからnodeで実行します。ただし、[RFC: React Server Module Conventions v2](https://github.com/reactjs/rfcs/pull/227)で説明されているように、このコードを実行するには`node --conditions react-server`というオプションを渡す必要があります。

実行すると、次のように出力されます。

```
J0:["$","div",null,{"children":[["$","h1",null,{"children":"React Server Components example"}],["$","p",null,{"children":"Hello, world!"}]]}]
```

すでにご存知の方もいるかもしれませんが、これはRSCで使われているプロトコルでレンダリング結果を表現したものです。実際のRSCのシチュエーションでは、この文字列がクライアント側のReactによって解釈されてレンダリングに使用されます。

これは一見するとJSXをJSONに変換しただけに見えますが、ちゃんとレンダリングされています。そのことを確かめるために、コンポーネントを追加してみましょう。

## コンポーネントを追加してみる

`App.tsx`を次のように2つに分割してみます。

```tsx:src/app/Page.tsx
import React from "react";

export const Page: React.FC<React.PropsWithChildren> = ({ children }) => {
  return (
    <div>
      <h1>React Server Components example</h1>
      {children}
    </div>
  );
};
```

```tsx:src/app/App.tsx
import { Page } from "./Page.js";

export const App: React.FC = () => {
  return (
    <Page>
      <p>Hello, world!</p>
    </Page>
  );
};
```

`App`コンポーネントから`Page`コンポーネントを使用するようにした形ですね。これで再びレンダリングを実行してみると、結果は次のようになります。

```
J0:["$","div",null,{"children":[["$","h1",null,{"children":"React Server Components example"}],["$","p",null,{"children":"Hello, world!"}]]}]
```

さっきと変わっていませんね。これは、`Page`や`App`がちゃんとサーバーサイドでレンダリングされて、ただのHTMLに解決されたことを意味しています。React Server Componentsでバンドルサイズが減るとよく言われますが、このように`Page`や`App`のレンダリングをサーバーサイドで解決することにより、これらのソースコードをクライアントに送る必要がないのがその理由です。

復習ですが、従来のいわゆるSSRでは、必ず「サーバー側でReactアプリをレンダリング→クライアント側でも同じアプリをレンダリングしてhydration」というステップを踏む必要があったため、アプリケーションの全体をクライアントに送る必要がありました。RSCではこの原則を廃し、コンポーネントを「サーバー用」と「クライアント用」に分類して後者のみをクライアントに送るのがポイントでした。このことが上の小さな例からでも実感できます。

## Server Componentではステートなどは使えない

現在のコードでは、ReactアプリケーションをServer Componentsの環境で動作させています。言い方を変えれば、`App`および`Page`はサーバー用のコンポーネントです。

サーバー用のコンポーネントはステートを持たないため、`useState`が使えません。他に`useEffect`なども使えず、サーバー用のコンポーネントで使えるフックは限定的です。このことを確かめてみましょう。

```tsx:src/app/Clock.tsx
import { useEffect, useState } from "react";

export const Clock: React.FC = () => {
  const [time, setTime] = useState("");
  useEffect(() => {
    const timerId = setInterval(() => {
      const now = new Date();
      const hours = now.getHours().toString().padStart(2, "0");
      const minutes = now.getMinutes().toString().padStart(2, "0");
      const seconds = now.getSeconds().toString().padStart(2, "0");
      setTime(`${hours}:${minutes}:${seconds}`);
    }, 1000);
    return () => {
      clearInterval(timerId);
    };
  }, []);

  return <p>{time}</p>;
};
```

```diff tsx:src/app/App.tsx
+import { Clock } from "./Clock.js";
 import { Page } from "./Page.js";

 export const App: React.FC = () => {
   return (
     <Page>
        <p>Hello, world!</p>
+      <Clock />
     </Page>
   );
 };
```

では、これを実行してレンダリングしたらどうなるか見てみましょう。……と言いたいところですが、実はレンダリングを試みることすらできません。次の実行時エラーが出ます。

```
import { useEffect, useState } from "react";
         ^^^^^^^^^
SyntaxError: Named export 'useEffect' not found. The requested module 'react' is a CommonJS module, which may not support all module.exports as named exports.
CommonJS modules can always be imported via the default export, for example using:

import pkg from 'react';
const { useEffect, useState } = pkg;
```

そもそも`react`から`useEffect`などをインポートすることすらできなくなってしまいました。

その理由は、`react`の`package.json`を見れば明らかになります。

```json:reactのpackage.json（抜粋）
  "exports": {
    ".": {
      "react-server": "./react.shared-subset.js",
      "default": "./index.js"
    },
    "./package.json": "./package.json",
    "./jsx-runtime": "./jsx-runtime.js",
    "./jsx-dev-runtime": "./jsx-dev-runtime.js"
  },
```

つまり、`react-server`環境下では`react.shared-subset.js`という別のエントリーポイントが読み込まれるようになっています。そして、調べてみると分かりますが、このエンドポイントからは`useEffect`や`useState`が提供されていないのです。これらのフックがServer Componentsでは使えないということが、node.jsのconditional exportsを使ってこのような形で表現されています。

ここまでの内容がリポジトリの`step2`に対応しています。次の章ではサーバー用コンポーネントとクライアント用コンポーネントの分類を実践してみましょう。