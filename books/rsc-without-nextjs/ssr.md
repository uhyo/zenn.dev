---
title: "SSRを実装する"
---

前章では、サーバー側のレンダリングとクライアント側のレンダリングを、HTMLの出力を通じて統合しました。

前章で行ったのは、SSRを伴わないRSCの実装でした。それでもRSCの利点は出ていますが、実際のところRSCはSSRと併用されることがほとんどです。そこで、この章ではSSRを実装してみます。

## RSCのレンダリングの実装を分ける

まず、RSCのレンダリングに特化したファイルを新たに用意し、`src/rsc.tsx`とします。改めて作ってみると、これくらい単純な実装になります。

```tsx:src/rsc.tsx
import type {} from "react/canary";
// @ts-expect-error
import rsdws from "react-server-dom-webpack/server";
const { renderToPipeableStream } = rsdws;

import { App } from "./app/server/App.js";
import { bundlerConfig } from "./app/server/Client.js";

const stream = renderToPipeableStream(<App />, bundlerConfig);

stream.pipe(process.stdout);
```

このファイルをコンパイル後Node.jsで実装してみます。

```sh
node --conditions react-server dist/rsc.js
```

すると標準出力にRSCプロトコルの文字列が出力されます。

```
M1:{"id":"__mod__","name":"Clock","chunks":[],"async":true}
J0:["$","div",null,{"children":[["$","h1",null,{"children":"React Server Components example"}],[["$","p",null,{"children":"Hello, world!"}],["$","@1",null,{}]]]}]
```

## SSRの実装

次に、`src/server.tsx`はSSRを担当するように直していきます。

全体の実装は長いので隠しておきます。

:::details 全体の実装

```tsx:src/server.tsx
import type {} from "react/canary";
import { spawn } from "child_process";
import { Readable, PassThrough } from "stream";
import type { ReadableStream } from "stream/web";
import { fileURLToPath } from "url";
import { createWriteStream } from "fs";

import ReactDOM from "react-dom/server";
// @ts-expect-error
import rsdwc from "react-server-dom-webpack/client";
const { createFromReadableStream } = rsdwc;

import { allClientComponents } from "./app/client/clientComponents.js";
import { use } from "react";

function render() {
  const rscFile = fileURLToPath(
    new URL("./rsc.js", import.meta.url).toString(),
  );
  const proc = spawn("node", ["--conditions", "react-server", rscFile], {
    stdio: ["ignore", "pipe", "inherit"],
  });
  return proc.stdout;
}

// @ts-expect-error
globalThis.__webpack_require__ = async () => {
  return allClientComponents;
};

async function renderHTML(): Promise<Readable> {
  const [stream1, stream2] = Readable.toWeb(render()).tee();

  const chunk = createFromReadableStream(stream1);
  const rscData = await readAll(stream2);

  const PageContainer: React.FC = () => {
    return (
      <html lang="en">
        <head>
          <meta charSet="utf-8" />
        </head>
        <body>
          <div id="app">{use(chunk)}</div>
          <script id="rsc-data" data-data={rscData} />
        </body>
      </html>
    );
  };

  const htmlStream = ReactDOM.renderToPipeableStream(<PageContainer />, {
    bootstrapModules: ["src/client.tsx"],
  }).pipe(new PassThrough());

  return htmlStream;
}

async function readAll(stream: ReadableStream<Uint8Array>): Promise<string> {
  let result = "";
  const decoder = new TextDecoder();
  for await (const chunk of stream) {
    result += decoder.decode(chunk);
  }
  return result;
}

renderHTML().then(async (html) => {
  const file = createWriteStream("./index.html");
  html.pipe(file);
});
```

:::

順番に見ていきましょう。まずは、`render`関数です。

```tsx
function render() {
  const rscFile = fileURLToPath(
    new URL("./rsc.js", import.meta.url).toString(),
  );
  const proc = spawn("node", ["--conditions", "react-server", rscFile], {
    stdio: ["ignore", "pipe", "inherit"],
  });
  return proc.stdout;
}
```

この関数は、別のNode.jsプロセスを起動し、そのプロセスの標準出力を返します。このプロセスは、先ほど作った`src/rsc.tsx`を実行します。結果として、`render()`が返すストリームは、RSCプロトコルの文字列になります。

ここでわざわざ別のプロセスとする理由は、`rsc.tsx`は`--conditions react-server`を指定して実行する必要がある一方、`server.tsx`は`--conditions react-server`無しで実行したいからです。

そのため、`npm start`を使っている場合はpackage.jsonを書き換えておきましょう。

```diff:json
     "build": "tsc",
-    "start": "node --conditions react-server dist/server.js",
+    "start": "node dist/server.js",
     "vite": "vite dev",
     "test": "echo \"Error: no test specified\" && exit 1"
```

次の箇所では`__webpack_require__`を設定していますが、これは`client.tsx`の場合と同様に実際のコンポーネント実装を与えています。今回はSSRであり、これはサーバー側でクライアントコンポーネントのレンダリングも行ってしまうことを指すからです。

```ts
// @ts-expect-error
globalThis.__webpack_require__ = async () => {
  return allClientComponents;
};
```

そして、SSRの本体はここです。

```tsx
async function renderHTML(): Promise<Readable> {
  const [stream1, stream2] = Readable.toWeb(render()).tee();

  const chunk = createFromReadableStream(stream1);
  const rscData = await readAll(stream2);

  const PageContainer: React.FC = () => {
    return (
      <html lang="en">
        <head>
          <meta charSet="utf-8" />
        </head>
        <body>
          <div id="app">{use(chunk)}</div>
          <script id="rsc-data" data-data={rscData} />
        </body>
      </html>
    );
  };

  const htmlStream = ReactDOM.renderToPipeableStream(<PageContainer />, {
    bootstrapModules: ["src/client.tsx"],
  }).pipe(new PassThrough());

  return htmlStream;
}
```

その場で`PageContainer`を定義していますが、今回は`html`要素から全部レンダリングしています。Reactのリファレンスによればこれがベストプラクティスのようです。`src/client.tsx`を読み込む`script`要素の生成も`bootstrapModules`によって行っています。

最後のパートは、`renderHTML`の結果をファイルに書き出す部分です。

```ts
renderHTML().then(async (html) => {
  const file = createWriteStream("./index.html");
  html.pipe(file);
});
```

これで、`npm start`を実行すると`index.html`が生成されます。内容は以下のようになります。読みやすいように整形しています。

```html
<!DOCTYPE html><html lang="en">
  <head>
    <meta charSet="utf-8"/>
  </head>
  <body>
    <div id="app"><div><h1>React Server Components example</h1><p>Hello, world!</p><p></p></div></div>
    <script id="rsc-data" data-data="M1:（中略）"></script>
    <script type="module" src="src/client.tsx" async=""></script>
  </body>
</html>
```

`bootstrapModules`に指定した内容がbody要素の最後に自動的に付加されていますね。

クライアント側も、今回はSSRなのでハイドレーションを行う必要があります。よって、`src/client.tsx`は少し修正します。

```diff:ts
-createRoot(app).render(<Container />);
+hydrateRoot(app, <Container />);
```

前章と同様に、先ほど出力された`index.html`はViteのエントリーポイントになるので、`npm run vite`で実行してブラウザで開くとハイドレーションが実行されて無事にReactアプリケーションが動作します。

以上でRSCとSSRの併用が実装できました。ここまでの内容を反映したブランチは`step6`です。

最後に、次の章でサーバーサイドでRSCの威力をもうちょっと実感してみましょう。