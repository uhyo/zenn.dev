---
title: "SSRでサーバーとクライアントを統合する"
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

## SSRの実装

今回はサーバーの出力結果をRSCプロトコルではなくHTMLにしてみましょう。

そのために、まず次の下準備が必要です。

```diff tsx:src/app/server/App.tsx
 import { Suspense } from "react";
 import { Client } from "./Client.js";
 import { Page } from "./Page.js";
 
 export const App: React.FC = () => {
   return (
     <Page>
       <p>Hello, world!</p>
+      <Suspense fallback={<p>Loading...</p>}>
         <Client.Clock />
+      </Suspense>
     </Page>
   );
 };
```

実は、サーバーからクライアント向けコンポーネントを使う場合、このように`Suspense`で囲むのが鉄則です。今回はSSRなのでのちのちhydrationすることになりますが、Suspenseは部分hydrationの境界となります。そもそも、前章までで見たようにクライアントコンポーネントは非同期で読み込むことが想定されているので、必然的にサスペンドします。その影響を受けない部分だけサーバーサイドでレンダリングするという意味を込めて、このようにSuspenseで囲みます。

それができたら、サーバーのエントリーポイントの実装を次のように書き換えます。

```tsx:src/server.tsx
import { Readable, PassThrough } from "stream";
import ReactDOM from "react-dom/server";
// @ts-expect-error
import rsdws from "react-server-dom-webpack/server";
const { renderToPipeableStream } = rsdws;
// @ts-expect-error
import rsdwc from "react-server-dom-webpack/client";
const { createFromReadableStream } = rsdwc;

import { App } from "./app/server/App.js";
import { bundlerConfig } from "./app/server/Client.js";
// @ts-expect-error
import { use } from "react";

const stream = Readable.toWeb(
  renderToPipeableStream(<App />, bundlerConfig).pipe(new PassThrough())
);

class ClientComponentError extends Error {}

const chunk = createFromReadableStream(stream);
// @ts-expect-error
globalThis.__webpack_require__ = async () => {
  throw new ClientComponentError("Client Component");
};

const Container = () => {
  return use(chunk);
};

ReactDOM.renderToPipeableStream(<Container />, {
  onError(error) {
    if (error instanceof ClientComponentError) {
      // 握りつぶす
      return;
    }
    console.error(error);
  },
}).pipe(process.stdout);
```

実装を順に解説します。

```ts
const stream = Readable.toWeb(
  renderToPipeableStream(<App />, bundlerConfig).pipe(new PassThrough())
);
```

ここは前章までと同じで、`react-dom-server-webpack/server`からインポートした`renderToPipeableStream`を用いてRSCのプロトコル（内部的にはReact Flightのプロトコルと言うらしいです）を出力します。今回はこれを、node.jsのstreamを経由してWHATWG Streamに変換しています。

```ts
class ClientComponentError extends Error {}

const chunk = createFromReadableStream(stream);
// @ts-expect-error
globalThis.__webpack_require__ = async () => {
  throw new ClientComponentError("Client Component");
};

const Container = () => {
  return use(chunk);
};
```

ここも前章と同じで、RSCのプロトコルの内容を出力するコンポーネントを用意しています。ただし、今回は`__webpack_require__`のダミー実装はエラーを発生させるようにしています。実はSSRの際は、Suspense境界の内側でエラーが発生した場合はそのSuspenseの中身はクライアント側でレンダリングが試みられるようになっています。これを利用し、サーバーサイドではクライアント用コンポーネントをレンダリングしないようにしています。

```ts
ReactDOM.renderToPipeableStream(<Container />, {
  onError(error) {
    if (error instanceof ClientComponentError) {
      // 握りつぶす
      return;
    }
    console.error(error);
  },
}).pipe(process.stdout);
```

ここがレンダリング結果をHTMLで出力するところです。SSR用のHTML出力APIは`react-dom/server`から得られる`renderToPipeableStream`なので、これを用います。`onError`では`__webpack_require__`から出るエラーを握りつぶしています。これは、デフォルトだとエラーがコンソールに出力されてしまうことに対する処置です。

以上のコードを実行すると、次の結果が出力されます。ちゃんとHTMLが出力されていますね。Suspense部分はフォールバックになっています。`template`内にエラーメッセージとかコールスタックが出力されていますが、これは`NODE_ENV=production`にすると消えるので問題ありません。`<!--$!-->`や`<!--/%-->`といったコメントがありますが、これらはSuspense境界をクライアント側に伝えるためのものだと思われます。

```html
<div><h1>React Server Components example</h1><p>Hello, world!</p><!--$!--><template data-msg="Client Component" data-stck="
    at Container (file:///Users/uhyo/private/rsc-without-nextjs/dist/server.js:23:12)"></template><p>Loading...</p><!--/$--></div>
```

ここまでの実装が`step5-2`ブランチにあります。

## HTMLを出力する

今回はSSRということで、ちゃんと完全なHTMLを出力するようにしましょう。これは普通にStreamの出力結果の前後のHTML文字列を足すだけでよく、こういう感じになります。

```tsx:src/server.tsx（抜粋）
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
    <div id="app">`;

  const htmlStream = ReactDOM.renderToPipeableStream(<Container />, {
    onError(error) {
      if (error instanceof ClientComponentError) {
        // 握りつぶす
        return;
      }
      console.error(error);
    },
  }).pipe(new PassThrough());
  for await (const chunk of htmlStream) {
    result += chunk;
  }

  result += `</div>
    <script type="module" src="src/client.tsx"></script>
  </body>
</html>`;
  return result;
}
```

これを動かせば、このようなHTMLが`index.html`として出力されるでしょう。うまくいっていますね。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
  </head>
  <body>
    <div id="app"><div><h1>React Server Components example</h1><p>Hello, world!</p><!--$!--><template data-msg="Client Component" data-stck="
    at Container (file:///Users/uhyo/private/rsc-without-nextjs/dist/server.js:24:12)"></template><p>Loading...</p><!--/$--></div></div>
    <script type="module" src="src/client.tsx"></script>
  </body>
</html>
```

ただし、今回はSSRということで、クライアントサイドでのhydrationが必要です。そのために、RSCプロトコルでの文字列も別途出力してあげる必要があります。その実装を足しましょう。

:::details 時間がある方は下の結果を見て自分で実装してみましょう

```ts:src/server.tsx
import { Readable, PassThrough } from "stream";
import { writeFile } from "fs/promises";
import ReactDOM from "react-dom/server";
// @ts-expect-error
import rsdws from "react-server-dom-webpack/server";
const { renderToPipeableStream } = rsdws;
// @ts-expect-error
import rsdwc from "react-server-dom-webpack/client";
const { createFromReadableStream } = rsdwc;

import { App } from "./app/server/App.js";
import { bundlerConfig } from "./app/server/Client.js";
// @ts-expect-error
import { use } from "react";

const [stream1, stream2] = Readable.toWeb(
  renderToPipeableStream(<App />, bundlerConfig).pipe(new PassThrough())
).tee();

class ClientComponentError extends Error {}

const chunk = createFromReadableStream(stream1);
// @ts-expect-error
globalThis.__webpack_require__ = async () => {
  throw new ClientComponentError("Client Component");
};

const Container = () => {
  return use(chunk);
};

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
    <div id="app">`;

  const htmlStream = ReactDOM.renderToPipeableStream(<Container />, {
    onError(error) {
      if (error instanceof ClientComponentError) {
        // 握りつぶす
        return;
      }
      console.error(error);
    },
  }).pipe(new PassThrough());
  for await (const chunk of htmlStream) {
    result += chunk;
  }

  result += `</div>
    <script id="ssr-data" type="text/plain" data-data="`;

  const decoder = new TextDecoder("utf-8");
  for await (const chunk of stream2) {
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

:::

こんな感じでscript要素にRSCプロトコルの文字列を突っ込みましょう。ちなみに、`script`の中身ではなく`data-data`属性に出力しているのは、そちらの方がエスケープ処理が楽だからです。

```diff html
 <!DOCTYPE html>
 <html lang="en">
   <head>
     <meta charset="utf-8" />
   </head>
   <body>
     <div id="app"><div><h1>React Server Components example</h1><p>Hello, world!</p><!--$!--><template data-msg="Client Component" data-stck="
     at Container (file:///Users/uhyo/private/rsc-without-nextjs/dist/server.js:24:12)"></template><p>Loading...</p><!--/$--></div></div>
+    <script id="ssr-data" type="text/plain" data-data="S1:&quot;react.suspense&quot;
+M2:{&quot;id&quot;:&quot;__mod__&quot;,&quot;name&quot;:&quot;Clock&quot;,&quot;chunks&quot;:[],&quot;async&quot;:true}
+J0:[&quot;$&quot;,&quot;div&quot;,null,{&quot;children&quot;:[[&quot;$&quot;,&quot;h1&quot;,null,{&quot;children&quot;:&quot;React Server Components example&quot;}],[[&quot;$&quot;,&quot;p&quot;,null,{&quot;children&quot;:&quot;Hello, world!&quot;}],[&quot;$&quot;,&quot;$1&quot;,null,{&quot;fallback&quot;:[&quot;$&quot;,&quot;p&quot;,null,{&quot;children&quot;:&quot;Loading...&quot;}],&quot;children&quot;:[&quot;$&quot;,&quot;@2&quot;,null,{}]}]]]}]
+"></script>
     <script type="module" src="src/client.tsx"></script>
   </body>
 </html>
```

## クライアントサイドの実装

さて、上で出力されたHTMLの下から3行目を見ると分かるように、クライアント側の実装は出力されたHTMLをViteに食わせることでやってしまおうという魂胆です。まずサーバーをビルド→次にクライアントをViteでビルドという流れですね。もう少し整えることもできますが、今回はハンズオンなのでこれくらいで勘弁してください。ということで、`src/client.tsx`を実装します。実装できる自信のある方は自分で実装してみてもいいかもしれません。

```tsx:src/client.tsx
// @ts-expect-error
import { createFromReadableStream } from "react-server-dom-webpack/client";
// @ts-expect-error
import { use } from "react";
import { hydrateRoot } from "react-dom/client";
import { Clock } from "./app/client/Clock.js";

const app = document.getElementById("app");
const ssrData = document.getElementById("ssr-data")?.getAttribute("data-data");

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

const Container = () => {
  return use(chunk);
};

hydrateRoot(app, <Container />);
```

一つ目のポイントは、`<script id="ssr-data">`の`data-data`属性から取得した文字列を`ReadableStream`にして`ssrDataStream`として`createFromReadableStream`に渡していることです。前と同じく`fetch`を経由してもよかったのですが、WHATWG Streamの練習にはちょうどいい課題なのでやってみました。

もう一つのポイントは`__webpack_require__`の実装です。今回は面倒なのでチャンク分割などは実装していませんので、全てのクライアント向けコンポーネントを含んだ`allClientComponents`オブジェクトをあらかじめ用意しておき、それを返す実装になっています。ちなみに、今回はサーバー側でモジュールのメタデータを`chunks: []`としていますのでチャンク読み込みが発生しません。そのため、`__webpack_chunk_load__`の実装は必要ありません。

以上の実装でViteを実行すれば、アプリが正常に動作していることが確かめられるはずです。HTMLに埋め込まれたRSCプロトコルの文字列をもとにクライアントでのレンダリングとhydrationが行われます。

このアプリでは、サーバーサイドのコンポーネント（`App`や`Page`）はクライアントに送られていない一方、クライアントサイドのコンポーネント（`Clock`）の実装はサーバーサイドは関知していません。つまり、1つのアプリを2か所で分担してレンダリングするということが達成できていますね。

成果物としては、従来のフレームワーク無しSSRにRSCを使ってみたという感じになりました。

ここまでの実装が`step5-3`ブランチにあります。

最後に、次の章でサーバーサイドでRSCの威力をもうちょっと実感して終わりにしましょう。