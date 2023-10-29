---
title: "サーバーとクライアントを統合する"
---

前章の実装では、サーバーとクライアントが完全に別々のものとして実装していました。これだと書きやすさの面でRSCの利点があまり活きていませんので、次は1つのReactアプリケーションでサーバーとクライアントの分担が実現できるようにしましょう。

実装にあたって問題となるのは、サーバー用のコンポーネントとクライアント用のコンポーネントをどう区別するのかです。今回はハンズオンなのであまりややこしい実装は避けたいと思います。そこで、クライアント用のコンポーネントはこのように`Client.Xxx`という形で使うことにします。

```tsx
export const App: React.FC = () => {
  return (
    <Page>
      <p>Hello, world!</p>
      <Client.Clock />
    </Page>
  );
};
```

## クライアント用コンポーネント管理基盤の準備

上のようにサーバーサイドで`Client.Clock`としてクライアント向けコンポーネントを使用できるようにするために、適当な管理基盤を準備します。まず、クライアント用コンポーネントを登録できる`ClientComponents`インターフェースを用意します。

```ts:src/app/shared/registry.ts
import React from "react";

declare global {
  interface ClientComponents {}
}

export type ClientComponent<Key extends keyof ClientComponents> = React.FC<
  ClientComponents[Key]
>;
```

クライアント用コンポーネントを定義する側はこのようにします。

```ts:src/app/client/Clock.tsx
import { useEffect, useState } from "react";
import { ClientComponent } from "../shared/registry.js";

declare global {
  interface ClientComponents {
    Clock: {};
  }
}

export const Clock: ClientComponent<"Clock"> = () => {
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

`declare global`の5行で`Clock`というクライアント用コンポーネントを定義し、`Client.Clock`で使えるようにしています。この一手間がちょっと面倒ですが、これを省くためにはいよいよバンドラに手を入れる必要が出てきてハンズオンに収まらなくなるので、今回はこれで妥協します。

以上の準備により、サーバーサイドではこのようにクライアント向けコンポーネントを使うことができます。まず`Client`オブジェクトを定義します。

```tsx:src/app/server/Client.ts
import { ClientComponent } from "../shared/registry.js";

export const Client = new Proxy(
  {},
  {
    get(_, key) {
      return {
        $$typeof: Symbol.for("react.module.reference"),
        filepath: "__mod__",
        name: key,
      };
    },
  }
) as {
  [K in keyof ClientComponents]: ClientComponent<K>;
};

export const bundlerConfig = {
  __mod__: new Proxy(
    {},
    {
      get(_, key) {
        return {
          id: "__mod__",
          name: key,
          chunks: [],
        };
      },
    }
  ),
};
```

次にそれを用いる`App`コンポーネントを実装します。

```tsx:src/app/server/App.tsx
// サーバー側の実装
import { Client } from "./Client.js";
import { Page } from "./Page.js";

export const App: React.FC = () => {
  return (
    <Page>
      <p>Hello, world!</p>
      <Client.Clock />
    </Page>
  );
};

```

最後にサーバーのエントリーポイントです。

```tsx:src/server.tsx
// @ts-expect-error
import rsdws from "react-server-dom-webpack/server";
const { renderToPipeableStream } = rsdws;

import { App } from "./app/server/App.js";
import { bundlerConfig } from "./app/server/Client.js";

renderToPipeableStream(<App />, bundlerConfig).pipe(process.stdout);
```

`Clock`と`bundlerConfig`の実装が手書きだったのを、Proxyを使ってちょっと汎用的にしました。型レベルではクライアント向けコンポーネントを`ClientComponents`インターフェースに登録するようにすることで心ばかりの型安全性を確保しました。

以上の実装を実行すると、前章までと同じような結果が得られます。

```
M1:{"id":"__mod__","name":"Clock","chunks":[]}
J0:["$","div",null,{"children":[["$","h1",null,{"children":"React Server Components example"}],[["$","p",null,{"children":"Hello, world!"}],["$","@1",null,{}]]]}]
```

ここまでの実装が`step5-1`ブランチにあります。

## HTMLを出力する

さて、ここまでの実装ではサーバーの出力結果がRSCプロトコルになっています。サーバーとクライアントをうまく統合するために、HTMLを出力してみましょう。

といってもここでは何か特別なことをするわけではなく、上述のRSCプロトコル文字列を含む、HTMLのガワを出力するだけです。コードを一気にお見せします。

```tsx:src/server.tsx
import type {} from "react/canary";
import { Readable, PassThrough } from "stream";
import { writeFile } from "fs/promises";
// @ts-expect-error
import rsdws from "react-server-dom-webpack/server";
const { renderToPipeableStream } = rsdws;

import { App } from "./app/server/App.js";
import { bundlerConfig } from "./app/server/Client.js";

const stream = Readable.toWeb(
  renderToPipeableStream(<App />, bundlerConfig).pipe(new PassThrough())
);

renderHTML().then(async (html) => {
  await writeFile("./index.html", html);
});

async function renderHTML() {
  let result = `<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
  </head>
  <body>
    <div id="app"></div>
    <script id="rsc-data" type="text/plain" data-data="`;

  const decoder = new TextDecoder("utf-8");
  for await (const chunk of stream) {
    result += escapeHTML(decoder.decode(chunk));
  }

  result += `"></script>
    <script type="module" src="src/client.tsx"></script>
  </body>
</html>`;
  return result;
}

const escapeHTMLTable: Partial<Record<string, string>> = {
  "<": "&lt;",
  ">": "&gt;",
  "&": "&amp;",
  "'": "&#x27;",
  '"': "&quot;",
};

function escapeHTML(str: string) {
  return str.replace(/[<>&'"]/g, (char) => escapeHTMLTable[char] ?? "");
}
```

こんな感じでscript要素にRSCプロトコルの文字列を突っ込みましょう。ちなみに、`script`の中身ではなく`data-data`属性に出力しているのは、そちらの方がエスケープ処理が楽だからです。出力されるHTMLはこんな具合です。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
  </head>
  <body>
    <div id="app"></div>
    <script id="rsc-data" type="text/plain" data-data="S1:&quot;react.suspense&quot;
+M2:{&quot;id&quot;:&quot;__mod__&quot;,&quot;name&quot;:&quot;Clock&quot;,&quot;chunks&quot;:[],&quot;async&quot;:true}
+J0:[&quot;$&quot;,&quot;div&quot;,null,{&quot;children&quot;:[[&quot;$&quot;,&quot;h1&quot;,null,{&quot;children&quot;:&quot;React Server Components example&quot;}],[[&quot;$&quot;,&quot;p&quot;,null,{&quot;children&quot;:&quot;Hello, world!&quot;}],[&quot;$&quot;,&quot;$1&quot;,null,{&quot;fallback&quot;:[&quot;$&quot;,&quot;p&quot;,null,{&quot;children&quot;:&quot;Loading...&quot;}],&quot;children&quot;:[&quot;$&quot;,&quot;@2&quot;,null,{}]}]]]}]
+"></script>
    <script type="module" src="src/client.tsx"></script>
  </body>
</html>
```

## クライアントサイドの実装

さて、上で出力されたHTMLの下から3行目を見ると分かるように、`.ts`ファイルを直接script要素に与えています。これは、クライアント側の実装は出力されたHTMLをViteに食わせることでやってしまおうという魂胆です。まずサーバーをビルド→次にクライアントをViteでビルドという流れですね。もう少し整えることもできますが、今回はハンズオンなのでこれくらいで勘弁してください。ということで、`src/client.tsx`を実装します。実装できる自信のある方は自分で実装してみてもいいかもしれません。

```tsx:src/client.tsx
// @ts-expect-error
import { createFromReadableStream } from "react-server-dom-webpack/client";
import { use } from "react";
import { createRoot } from "react-dom/client";
import { Clock } from "./app/client/Clock.js";

const app = document.getElementById("app");
const ssrData = document.getElementById("rsc-data")?.getAttribute("data-data");

if (app === null) {
  throw new Error("Root element does not exist");
}

if (ssrData == null) {
  throw new Error("ssrData is not provided");
}

const { readable: ssrDataStream, writable } = new TransformStream<
  Uint8Array,
  Uint8Array
>();

(async () => {
  const encoder = new TextEncoder();
  const writer = writable.getWriter();
  await writer.write(encoder.encode(ssrData));
  await writer.close();
})();

const allClientComponents: {
  [K in keyof ClientComponents]: React.FC<ClientComponents[K]>;
} = {
  Clock,
};

const chunk = createFromReadableStream(ssrDataStream);
// @ts-expect-error
globalThis.__webpack_require__ = async () => {
  return allClientComponents;
};

const Container: React.FC = () => {
  return use(chunk);
};

createRoot(app).render(<Container />);
```

一つ目のポイントは、`<script id="rsc-data">`の`data-data`属性から取得した文字列を`ReadableStream`にして`ssrDataStream`として`createFromReadableStream`に渡していることです。前と同じく`fetch`を経由してもよかったのですが、WHATWG Streamの練習にはちょうどいい課題なのでやってみました。

もう一つのポイントは`__webpack_require__`の実装です。今回は面倒なのでチャンク分割などは実装していませんので、全てのクライアント向けコンポーネントを含んだ`allClientComponents`オブジェクトをあらかじめ用意しておき、それを返す実装になっています。ちなみに、今回はサーバー側でモジュールのメタデータを`chunks: []`としていますのでチャンク読み込みが発生しません。そのため、`__webpack_chunk_load__`の実装は必要ありません。

これを実際に実行してみるには、まずサーバー側（`src/server.ts`）を実行します（サンプルリポジトリでは、`npm run build`の後に`npm start`）。そうしたら、`index.html`が出力されてそれがViteのエントリーポイントとなるので、`npm run vite`でViteを実行すれば結果が確認できます。

Viteを実行すると、アプリが正常に動作していることが確かめられるはずです。HTMLに埋め込まれたRSCプロトコルの文字列をもとにクライアントでのレンダリングが行われます。

このアプリでは、サーバーサイドのコンポーネント（`App`や`Page`）はクライアントに送られていない一方、クライアントサイドのコンポーネント（`Clock`）の実装はサーバーサイドは関知していません。つまり、1つのアプリを2か所で分担してレンダリングするということが達成できていますね。

以上が、RSCを用いた最も基本的なサーバーとクライアントの統合方法です。ここではサーバー側の出力がRSCプロトコルでありHTMLをサーバー側で生成していないので、SSRをせずにRSCを用いたことになります。

ここまでの実装が`step5-2`ブランチにあります。

次の章では、RSCとSSRの併用を試してみます。