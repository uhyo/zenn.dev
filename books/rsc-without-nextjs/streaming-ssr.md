---
title: "Streaming SSRの実装"
---

前章ではSSRの実装に成功しましたが、ストリーミングは特に行われていませんでした。

それは出力先が`index.html`なのでストリーミングの意味が特にないという理由もありますが、実装にもその原因がありました。というのも、前章の実装では`rsc-data`にRSCプロトコルの出力結果を全部書き出していたので、ストリーミングがその時点で終わっていたのです。

## 一旦ハイドレーションを消してみる

そこで、一旦ハイドレーションを消してみましょう。

```diff tsx
 async function renderHTML(): Promise<Readable> {
   const [stream1, stream2] = Readable.toWeb(render()).tee();

   const chunk = createFromReadableStream(stream1);
-  const rscData = await readAll(stream2);
+  // const rscData = await readAll(stream2);

   const PageContainer: React.FC = () => {
     return (
       <html lang="en">
         <head>
           <meta charSet="utf-8" />
         </head>
         <body>
           <div id="app">{use(chunk)}</div>
-          <script id="rsc-data" data-data={rscData} />
+          {/* <script id="rsc-data" data-data={rscData} /> */}
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

さらに、変化を観察するにはもう一箇所修正が必要です。`src/server/App.tsx`で、`Uhyo`をSuspenseで囲みます。

```diff tsx:src/server/App.tsx
 export const App: React.FC = () => {
   return (
     <Page>
       <p>Hello, world!</p>
+      <Suspense fallback={<p>Loading...</p>}>
         <Uhyo />
+      </Suspense>
       <Client.Clock />
     </Page>
   );
 };
```

以上の変更を加えて`npm start`を実行すると、次のような結果が得られます（例によって整形済）。

```html:index.html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charSet="utf-8"/>
  </head>
  <body>
    <div id="app">
      <div>
        <h1>React Server Components example</h1>
        <p>Hello, world!</p>
        <!--$?--><template id="B:0"></template><p>Loading...</p><!--/$-->
        <p></p>
      </div>
    </div>
    <script type="module" src="src/client.tsx" async=""></script>
    <div hidden id="S:0"><section><h2>Hello, I&#x27;m <!-- -->uhyo</h2><p>I am <!-- -->29<!-- --> years old.</p><p>My favorite languages are:</p><ul><li>TypeScript</li><li>Rust</li></ul></section></div>
    <script>$RC=function(b,c,e){c=document.getElementById(c);c.parentNode.removeChild(c);var a=document.getElementById(b);if(a){b=a.previousSibling;if(e)b.data="$!",a.setAttribute("data-dgst",e);else{e=b.parentNode;a=b.nextSibling;var f=0;do{if(a&&8===a.nodeType){var d=a.data;if("/$"===d)if(0===f)break;else f--;else"$"!==d&&"$?"!==d&&"$!"!==d||f++}d=a.nextSibling;e.removeChild(a);a=d}while(a);for(;c.firstChild;)e.insertBefore(c.firstChild,a);b.data="$"}b._reactRetry&&b._reactRetry()}};;$RC("B:0","S:0")</script>
  </body>
</html>
```

注目すべき点は、最初の出力において`<Uhyo />`の部分が`<p>Loading...</p>`に置き換わっていることです。これはSuspenseの機能によるものです。

`Uhyo`は非同期コンポーネントでありレンダリングに時間がかかるので、Suspenseがそれを検知して`fallback`を表示しています。

その後、`<div hidden id="S:0">`以降の何やら謎のHTMLが出力されています。これがストリーミングSSRの機能であり、`<Uhyo />`のレンダリングが完了したら、その結果を`template id="B:0">`の部分に差し込むことで、`Uhyo`のレンダリング結果をHTMLに反映します。

ここまでの内容が`step8-1`ブランチにあります。

## HTTPサーバーを建てる

出力先が`index.html`だとストリーミングの意味がないので、今回はHTTPサーバーを建ててストリーミングを確認してみましょう。`src/server.tsx`を編集して、ファイルに出力する代わりにこんな感じでサーバーを建てます。

```ts
const server = http.createServer((req, res) => {
  renderHTML()
    .then((html) => {
      res.writeHead(200, {
        "Content-Type": "text/html",
      });
      html.pipe(res);
    })
    .catch((error) => {
      console.error(error);
      res.writeHead(500, {
        "Content-Type": "text/plain",
      });
      res.end(String(error));
    });
});
server.listen(8888, () => {
  console.log("Listening on http://localhost:8888");
});
```

これを実行してブラウザで開くと、SSRの結果が得られるでしょう（クライアントのJavaScriptは動作しませんが）。

ストリーミングをわかりやすく体感するために、`Uhyo`がレンダリングに1秒かかるように修正してみましょう。

```diff tsx:src/server/Uhyo.tsx
 export const Uhyo: React.FC = async () => {
   const uhyoData = (
     await import("../../data/uhyo.json", {
       assert: {
         type: "json",
       },
     })
   ).default;
+ await new Promise((resolve) => setTimeout(resolve, 1000));
   return (
     <section>
       <h2>Hello, I'm {uhyoData.name}</h2>
       <p>I am {uhyoData.age} years old.</p>
       <p>My favorite languages are:</p>
       <ul>
         {uhyoData.favoriteLanguages.map((lang, i) => (
           <li key={i}>{lang}</li>
         ))}
       </ul>
     </section>
   );
 };
```

これでブラウザを開くと、しっかり1秒間「Loading...」が表示されてから`Uhyo`の内容が表示されるはずです。これがStreaming SSRによるものです。

ここまでの内容が`step8-2`ブランチにあります。

## ストリーミングとハイドレーションを両立する

ここからが仕上げです。ここまではストリーミングのために一旦ハイドレーションを切っていましたが、何とかハイドレーションを復活させましょう。

### Viteの開発サーバーを使う

ハイドレーションのためにはクライアント側でJavaScriptが動くようにする必要があります。ストリーミングと両立させるために、今回はViteの開発サーバーを`server.tsx`から呼び出す方法にしましょう。まず`vite.config.js`をそれ用に修正します。

```js:vite.config.js
import { defineConfig } from "vite";

export default defineConfig({
  server: {
    middlewareMode: true,
  },
  appType: "custom",
});
```

そして、server.tsxのHTTPサーバーにViteのミドルウェアを噛ませます。

```ts:src/server.tsx
import { createServer } from "vite";

// ...

const vite = await createServer();

const server = http.createServer((req, res) => {
  vite.middlewares(req, res, () => {
    renderHTML()
      .then((html) => {
        res.writeHead(200, {
          "Content-Type": "text/html",
        });
        html.pipe(res);
      })
      .catch((error) => {
        console.error(error);
        res.writeHead(500, {
          "Content-Type": "text/plain",
        });
        res.end(String(error));
      });
  });
});
server.listen(8888, () => {
  console.log("Listening on http://localhost:8888");
});
```

これで、`<script src="src/client.tsx">`が再び動作するようになります。ただし、RSCプロトコルの文字列を送るのをやめたので `ssrData is not provided`というエラーが出てハイドレーションはされません。

### RSCプロトコルの文字列をストリーミングする

ハイドレーションのためにはRSCプロトコルの文字列をクライアントに送る必要があります。しかし、Streaming SSRではHTML自体もストリーミングしています。HTMLのストリーミングが全部終わってから完成したRSC文字列をクライアントに送ることが考えられますが、そうするとハイドレーションが遅れてしまいます。

HTMLのストリーミングを活かしつつ、ハイドレーションもなるべく早く行うことでReactの特徴を最大限活かすことができます。そこで、クライアントが受け取るRSC文字列自体もストリーミング可能である（3つ前の章で`createFromReadableStream`していたことを思い出しましょう）ことを活かして、ストリーミング中のHTMLにRSC文字列を混ぜ込んでいきましょう。

具体的には、ReactのストリーミングSSRにおいて追加のコンテンツを表示するときはscript要素を配信することで行なっていたので、それを真似しましょう。追加のデータが来るたびに、それを表すscript要素を出力します。

RSCデータ（`render()`の結果）はすでにストリームなので、生のデータをscript要素に変換する`TransformStream`を実装してみましょう。

```ts
export class RscScriptStream extends TransformStream<string, string> {
  constructor() {
    super({
      transform(chunk, controller) {
        controller.enqueue(
          `<script>globalThis.rscData.push(${JSON.stringify(
            chunk,
          )});</script>\n`,
        );
      },
      flush(controller) {
        controller.enqueue(`<script>globalThis.rscData.end?.();</script>\n`);
      },
    });
  }
}
```

このように、`rscData`というグローバル変数にデータを追加していきます。

他に2つほどヘルパーとして、データを1行1チャンクに整形する`LinesStream`と、2つのストリームを並列に合成する`MergedStream`を実装します。

:::details 実装

```ts
export class LinesStream extends TransformStream<Uint8Array, string> {
  constructor() {
    let partialLine = "";
    const decoder = new TextDecoder();
    super({
      transform(chunk, controller) {
        const lines = decoder.decode(chunk).split("\n");
        const lastLine = lines.pop();
        if (lastLine === undefined) {
          return;
        }
        if (lines[0] === undefined) {
          partialLine += lastLine;
          return;
        }
        lines[0] = partialLine + lines[0];
        for (const line of lines) {
          controller.enqueue(line);
        }
        partialLine = lastLine;
      },
      flush(controller) {
        if (partialLine !== "") {
          controller.enqueue(partialLine);
        }
      },
    });
  }
}

export class MergedStream<T> extends ReadableStream<T> {
  constructor(streams: readonly ReadableStream<T>[], firstStream: number) {
    const readers = streams.map((stream) => stream.getReader());
    const readings: (Promise<[T, number]> | undefined)[] = [];
    const ends: (() => void)[] = [];
    const endedP = Promise.all(
      streams.map((stream) => {
        return new Promise<void>((resolve) => {
          ends.push(resolve);
        });
      }),
    );
    let firstSent = false;

    super({
      start(controller) {
        endedP.then(() => {
          controller.close();
        });
      },
      pull(controller) {
        return Promise.race(
          readers.map((reader, index): Promise<[T, number]> => {
            if (!firstSent && index !== firstStream) {
              return new Promise<never>(() => {});
            }
            const reading = readings[index];
            if (reading !== undefined) {
              return reading;
            }
            return (readings[index] = reader.read().then(({ value, done }) => {
              if (done) {
                reader.releaseLock();
                ends[index]();
                return new Promise<never>(() => {});
              }
              return [value, index];
            }));
          }),
        ).then(([value, index]) => {
          controller.enqueue(value);
          readings[index] = undefined;
          if (index === firstStream) {
            firstSent = true;
          }
        });
      },
      async cancel(reason) {
        await Promise.all(
          readers.map((reader) => {
            return reader.cancel(reason);
          }),
        );
      },
    });
  }
}
```

:::

できたら、`server.tsx`に組み込んでいきます。今度は`renderHTML`を直していきます。ReactのSSRのストリーム（`renderToPipeableStream`）と、RSC文字列のストリーム（`stream2`）を`MergedStream`で合成して返します。都合により返り値は`Readable`（Node.jsのストリーム）ではなく`ReadableStream`（Webのストリーム）になっています。

```tsx
async function renderHTML(): Promise<ReadableStream> {
  const [stream1, stream2] = Readable.toWeb(render()).tee();

  const chunk = createFromReadableStream(stream1);

  const PageContainer: React.FC = () => {
    return (
      <html lang="en">
        <head>
          <meta charSet="utf-8" />
        </head>
        <body>
          <div id="app">{use(chunk)}</div>
        </body>
      </html>
    );
  };

  return new Promise((resolve) => {
    const reactStream = ReactDOM.renderToPipeableStream(<PageContainer />, {
      bootstrapScriptContent: `
globalThis.rscData = [];
`,
      bootstrapModules: ["src/client.tsx"],
      onShellReady() {
        const rscStream = stream2
          .pipeThrough(new LinesStream())
          .pipeThrough(new RscScriptStream());

        const htmlStream = new MergedStream(
          [Readable.toWeb(reactStream.pipe(new PassThrough())), rscStream],
          0,
        );
        resolve(htmlStream);
      },
    });
  });
}
```

他のポイントとしては、`bootstrapScriptContent`に`globalThis.rscData = [];`を加えており、クライアント側で`rscData`が定義されるようにしています。

仕上げにHTTPサーバーの側で`renderHTML`の型の変化に対応します。

```diff ts
 const server = http.createServer((req, res) => {
   vite.middlewares(req, res, () => {
     renderHTML()
       .then((html) => {
         res.writeHead(200, {
           "Content-Type": "text/html",
         });
-        html.pipe(res);
+        Readable.fromWeb(html).pipe(res);
       })
       .catch((error) => {
         console.error(error);
         res.writeHead(500, {
           "Content-Type": "text/plain",
         });
         res.end(String(error));
       });
   });
 });
```

ここまで実装すれば、`npm start`でサーバーを起動してブラウザで開くと、ストリーミングされるHTMLにRSC文字列が混ぜ込まれているはずです。

```html

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charSet="utf-8"/>
  </head>
  <body>
    <div id="app"><div><h1>React Server Components example</h1><p>Hello, world!</p><!--$?--><template id="B:0"></template><p>Loading...</p><!--/$--><p></p></div></div>
    <script>globalThis.rscData = [];</script>
    <script type="module" src="src/client.tsx" async=""></script>
    <script>globalThis.rscData.push("S1:\"react.suspense\"");</script>
    <script>globalThis.rscData.push("M3:{\"id\":\"__mod__\",\"name\":\"Clock\",\"chunks\":[],\"async\":true}");</script>
    <script>globalThis.rscData.push("J0:[\"$\",\"div\",null,{\"children\":[[\"$\",\"h1\",null,{\"children\":\"React Server Components example\"}],[[\"$\",\"p\",null,{\"children\":\"Hello, world!\"}],[\"$\",\"$1\",null,{\"fallback\":[\"$\",\"p\",null,{\"children\":\"Loading...\"}],\"children\":\"@2\"}],[\"$\",\"@3\",null,{}]]]}]");</script>
    <script>globalThis.rscData.push("J2:[\"$\",\"section\",null,{\"children\":[[\"$\",\"h2\",null,{\"children\":[\"Hello, I'm \",\"uhyo\"]}],[\"$\",\"p\",null,{\"children\":[\"I am \",29,\" years old.\"]}],[\"$\",\"p\",null,{\"children\":\"My favorite languages are:\"}],[\"$\",\"ul\",null,{\"children\":[[\"$\",\"li\",\"0\",{\"children\":\"TypeScript\"}],[\"$\",\"li\",\"1\",{\"children\":\"Rust\"}]]}]]}]");</script>
    <div hidden id="S:0"><section><h2>Hello, I&#x27;m <!-- -->uhyo</h2><p>I am <!-- -->29<!-- --> years old.</p><p>My favorite languages are:</p><ul><li>TypeScript</li><li>Rust</li></ul></section></div>
    <script>$RC=function(b,c,e){c=document.getElementById(c);c.parentNode.removeChild(c);var a=document.getElementById(b);if(a){b=a.previousSibling;if(e)b.data="$!",a.setAttribute("data-dgst",e);else{e=b.parentNode;a=b.nextSibling;var f=0;do{if(a&&8===a.nodeType){var d=a.data;if("/$"===d)if(0===f)break;else f--;else"$"!==d&&"$?"!==d&&"$!"!==d||f++}d=a.nextSibling;e.removeChild(a);a=d}while(a);for(;c.firstChild;)e.insertBefore(c.firstChild,a);b.data="$"}b._reactRetry&&b._reactRetry()}};;$RC("B:0","S:0")</script>
  </body>
</html>
<script>globalThis.rscData.end?.();</script>
```

このように、従来のストリーミングSSRの内容に加えて、`globalThis.rscData`にRSC文字列が追加するスクリプトが配信されていることが分かりますね。

### クライアント側でハイドレーションを行う

では、最後にハイドレーションを復活させましょう。`src/client.tsx`で、`globalThis.rscData`からRSC文字列を読み取ってハイドレーションに使えばよいです。

これは意外と簡単で、`ssrDataStream`を作るところをこのように書き換えます。

```ts
declare global {
  // injected by server
  let rscData: string[];
}

const { readable: ssrDataStream, writable } = new TransformStream<
  Uint8Array,
  Uint8Array
>();

(() => {
  const encoder = new TextEncoder();
  const writer = writable.getWriter();

  const initialSsrData = rscData;
  rscData = {
    push(chunk: string) {
      writer.write(encoder.encode(chunk + "\n"));
    },
    end() {
      writer.close();
    },
  } as unknown as string[];

  writer.write(encoder.encode(initialSsrData.join("\n") + "\n"));
})();
```

これは何をやっているかお分かりでしょうか。このスクリプトが実行された時点で存在していた`rscData`は即座にストリームに突っ込みつつ、それ以降にサーバーから配信されて`rscData`に追加される分も逃さないように、`rscData`を書き換えて`push`がストリームに繋がるようにしています。

こうすることで、サーバーからまだ全てのRSC文字列が配信されていなくても、クライアント側でハイドレーションを開始できます。

これでブラウザで開くと、ストリーミングされるHTMLにハイドレーションが適用されるはずです。分かりやすくするために、`Uhyo`の待ち時間を5秒くらいに伸ばしましょう。

```diff tsx:src/server/Uhyo.tsx
 export const Uhyo: React.FC = async () => {
   const uhyoData = (
     await import("../../data/uhyo.json", {
       assert: {
         type: "json",
       },
     })
   ).default;
-  await new Promise((resolve) => setTimeout(resolve, 1000));
+  await new Promise((resolve) => setTimeout(resolve, 5000));
   return (
     <section>
       <h2>Hello, I'm {uhyoData.name}</h2>
       <p>I am {uhyoData.age} years old.</p>
       <p>My favorite languages are:</p>
       <ul>
         {uhyoData.favoriteLanguages.map((lang, i) => (
           <li key={i}>{lang}</li>
         ))}
       </ul>
     </section>
   );
 };
```

ブラウザで開くと、このように表示される時間があります。

![Loading...の下に13:03:18と表示されている](/images/rsc-without-nextjs/screenshot3.png)

この状態は、Loading...と表示されているのでストリーミングSSRがまだ途中であることを表しています。しかし、`Clock`が動作しています。`Clock`が動作しているということは`useEffect`が実行されているということなので、ハイドレーションがされていることを意味します。

これで、ストリーミングSSRをしながらそれが完了する前にハイドレーションを行うことができました。自前の実装でもこういうことができるのは嬉しいですね。

ここまでの実装が`step8-3`ブランチにあります。

### 後からストリーミングされたコンポーネントもハイドレーションできる

もう少し複雑なこともしてみましょう。まずはカウンターのクライアントコンポーネントを用意します。

```tsx:src/client/Counter.tsx
import { useState } from "react";

declare global {
  interface ClientComponents {
    Counter: {};
  }
}

export const Counter: React.FC = () => {
  const [count, setCount] = useState(0);
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount((c) => c + 1)}>Increment</button>
    </div>
  );
};
```

忘れずに`allClientComponents`の定義に加えます。

```diff tsx
 export const allClientComponents: {
   [K in keyof ClientComponents]: React.FC<ClientComponents[K]>;
 } = {
   Clock,
+  Counter,
 };
```

そして、このコンポーネントは`Uhyo`と同じ`Suspense`の下に置きましょう。つまり、`Uhyo`がレンダリング完了してから表示されるクライアントコンポーネントです。

```diff tsx:src/server/App.tsx
 export const App: React.FC = () => {
   return (
     <Page>
       <p>Hello, world!</p>
       <Suspense fallback={<p>Loading...</p>}>
         <Uhyo />
+        <Client.Counter />
       </Suspense>
       <Client.Clock />
     </Page>
   );
 };
```

これでブラウザを開くと、5秒待ってから`Uhyo`が表示され、それと同時に`Counter`が表示されるはずです。そして、`Counter`のボタンを押すとちゃんとステートが変化します。これは、ストリーミングされて後からレンダリングされたコンポーネントもハイドレーションできていることを表しています。インクリメンタルなハイドレーションができているということですね。

![Counterが表示され、数値が8となっているスクリーンショット](/images/rsc-without-nextjs/screenshot4.png)

疑り深い人はわざとミスマッチを発生させてみましょう。

```diff tsx:src/client/Counter.tsx
 export const Counter: React.FC = () => {
-  const [count, setCount] = useState(0);
+  const [count, setCount] = useState(Math.floor(Math.random() * 10));
   return (
     <div>
       <p>Count: {count}</p>
       <button onClick={() => setCount((c) => c + 1)}>Increment</button>
     </div>
   );
 };
```

そうすると、ページを開いてから5秒後にコンソールにハイドレーションミスマッチのエラーが発生します。これは、ハイドレーションが行われていることの証拠になりますね。

ここまでの実装は`step8-4`ブランチにあります。

## まとめ

この章では、ストリーミングSSRを実装し、ハイドレーションも最適な形で行われるようにしました。フレームワークが無くてもちゃんと作ればこれくらいのことができるのですね。

ちなみに、後半の`rscData.push`を出力する仕組みは、実はNext.jsも実際に似た感じの仕組みを採用しています（出力されるHTMLを覗いてみましょう）。
