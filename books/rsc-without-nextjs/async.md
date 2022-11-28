---
title: "サーバーサイドでasyncコンポーネントを使ってみる"
---

RSCの強力なところは、サーバーサイドならasync関数をコンポーネントとして使えることです。ということで、今回のハンズオンでもこれを試してみましょう。

## データを用意する

典型的な非同期処理のひとつはデータの読み込みです。ということで、今回は適当なJSONファイルを用意します。

```json:src/data/uhyo.json
{
  "name": "uhyo",
  "age": 28,
  "favoriteLanguages": [
    "TypeScript",
    "Rust"
  ]
}
```

## コンポーネントを用意して使ってみる

そして、それを読み込んで表示するコンポーネントを追加します。今回はdynamic importとともに`await`を使用しています。

```tsx:src/app/server/Uhyo.tsx
export const Uhyo = (async () => {
  const uhyoData = (
    await import("../../data/uhyo.json", {
      assert: { type: "json" },
    })
  ).default;
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
}) as unknown as React.FC;
```

このコンポーネントを`App.tsx`から使います。`async`コンポーネントは必然的にサスペンドするので、忘れずに`Suspense`で囲みます。

```diff tsx:src/app/server/App.tsx
+import { Suspense } from "react";
 import { Client } from "./Client.js";
 import { Page } from "./Page.js";
+import { Uhyo } from "./Uhyo.js";
 
 export const App: React.FC = () => {
   return (
     <Page>
       <p>Hello, world!</p>
+      <Suspense fallback={null}>
+        <Uhyo />
+      </Suspense>
       <Suspense fallback={<p>Loading...</p>}>
         <Client.Clock />
       </Suspense>
     </Page>
   );
 };
```

以上の変更でちゃんとasyncなコンポーネントが動作するはずです。サーバーを実行して`index.html`を出力→Viteで実行、という流れをとると以下のようにアプリが動作します。

![修正後のアプリのスクリーンショット](/images/rsc-without-nextjs/screenshot2.png)

ここで出力されているHTMLを見てみると、次のようになっています。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
  </head>
  <body>
    <div id="app"><div><h1>React Server Components example</h1><p>Hello, world!</p><!--$?--><template id="B:0"></template><!--/$--><!--$!--><template data-msg="Client Component" data-stck="
    at Container (file:///Users/uhyo/private/rsc-without-nextjs/dist/server.js:24:12)"></template><p>Loading...</p><!--/$--></div><div hidden id="S:0"><section><h2>Hello, I&#x27;m <!-- -->uhyo</h2><p>I am <!-- -->28<!-- --> years old.</p><p>My favorite languages are:</p><ul><li>TypeScript</li><li>Rust</li></ul></section></div><script>$RC=function(b,c,e){c=document.getElementById(c);c.parentNode.removeChild(c);var a=document.getElementById(b);if(a){b=a.previousSibling;if(e)b.data="$!",a.setAttribute("data-dgst",e);else{e=b.parentNode;a=b.nextSibling;var f=0;do{if(a&&8===a.nodeType){var d=a.data;if("/$"===d)if(0===f)break;else f--;else"$"!==d&&"$?"!==d&&"$!"!==d||f++}d=a.nextSibling;e.removeChild(a);a=d}while(a);for(;c.firstChild;)e.insertBefore(c.firstChild,a);b.data="$"}b._reactRetry&&b._reactRetry()}};;$RC("B:0","S:0")</script></div>
    <script id="ssr-data" type="text/plain" data-data="S1:&quot;react.suspense&quot;
M3:{&quot;id&quot;:&quot;__mod__&quot;,&quot;name&quot;:&quot;Clock&quot;,&quot;chunks&quot;:[],&quot;async&quot;:true}
J0:[&quot;$&quot;,&quot;div&quot;,null,{&quot;children&quot;:[[&quot;$&quot;,&quot;h1&quot;,null,{&quot;children&quot;:&quot;React Server Components example&quot;}],[[&quot;$&quot;,&quot;p&quot;,null,{&quot;children&quot;:&quot;Hello, world!&quot;}],[&quot;$&quot;,&quot;$1&quot;,null,{&quot;fallback&quot;:null,&quot;children&quot;:&quot;@2&quot;}],[&quot;$&quot;,&quot;$1&quot;,null,{&quot;fallback&quot;:[&quot;$&quot;,&quot;p&quot;,null,{&quot;children&quot;:&quot;Loading...&quot;}],&quot;children&quot;:[&quot;$&quot;,&quot;@3&quot;,null,{}]}]]]}]
J2:[&quot;$&quot;,&quot;section&quot;,null,{&quot;children&quot;:[[&quot;$&quot;,&quot;h2&quot;,null,{&quot;children&quot;:[&quot;Hello, I&#x27;m &quot;,&quot;uhyo&quot;]}],[&quot;$&quot;,&quot;p&quot;,null,{&quot;children&quot;:[&quot;I am &quot;,28,&quot; years old.&quot;]}],[&quot;$&quot;,&quot;p&quot;,null,{&quot;children&quot;:&quot;My favorite languages are:&quot;}],[&quot;$&quot;,&quot;ul&quot;,null,{&quot;children&quot;:[[&quot;$&quot;,&quot;li&quot;,&quot;0&quot;,{&quot;children&quot;:&quot;TypeScript&quot;}],[&quot;$&quot;,&quot;li&quot;,&quot;1&quot;,{&quot;children&quot;:&quot;Rust&quot;}]]}]]}]
"></script>
    <script type="module" src="src/client.tsx"></script>
  </body>
</html>
```

結果は想定通りですが、`<script>$RC=...`という謎のスクリプトが挟まっています。実はこれはSSRのSuspenseサポートにより出力されたもので、`<Uhyo />`を囲んでいる`Suspense`について最初はフォールバックを出力し、`Uhyo`のレンダリングが完了したらその結果でHTMLを差し替えるというインラインスクリプトが出力されています。これは本当のSSRであればありがたい機能で、サーバーサイドのコンポーネントがサスペンドしている最中であっても`Suspense`の外側はもうユーザーに送ることができます。

しかし今回はプリレンダリングなのでこの機能は不要ですね。

## 出力を調整する

`react-dom`の`renderToPipeableStream`はこのユースケースにも対応しています。具体的には、`renderHTML`を次のように書き換えましょう。

```tsx:src/server.tsx（抜粋）
async function renderHTML() {
  let result = `<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
  </head>
  <body>
    <div id="app">`;

  const htmlStream = new PassThrough();

  const reactHtmlStream = ReactDOM.renderToPipeableStream(<Container />, {
    onError(error) {
      if (error instanceof ClientComponentError) {
        // 握りつぶす
        return;
      }
      console.error(error);
    },
    onAllReady() {
      reactHtmlStream.pipe(htmlStream);
    },
  });
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
```

ポイントは、これまで`ReactDOM.renderToPipeableStream`の返り値をすぐに`pipe`していましたが、今回はそれを`onAllReady`コールバックが呼び出されてから`pipe`するように変更したところです。

この`onAllReady`というコールバックは全てのSuspenseが解消されたら呼び出されます。このタイミングを待ってから`pipe`することで、サスペンド中の状態が出力されるHTMLに現れず、結果のみが出力されます。実際に出力されたHTMLを見てみると、`Uhyo`のレンダリング結果が最初からベタ書きになっていることが分かります。これできれいなプリレンダリング結果になりました。

```html
<div id="app"><div><h1>React Server Components example</h1><p>Hello, world!</p><!--$--><section><h2>Hello, I&#x27;m <!-- -->uhyo</h2><p>I am <!-- -->28<!-- --> years old.</p><p>My favorite languages are:</p><ul><li>TypeScript</li><li>Rust</li></ul></section><!--/$--><!--$!--><template data-msg="Client Component" data-stck="
at Container (file:///Users/uhyo/private/rsc-without-nextjs/dist/server.js:24:12)"></template><p>Loading...</p><!--/$--></div></div>
```

ここまでの内容は`step6`ブランチにあります。