---
title: "サーバーサイドでasyncコンポーネントを使ってみる"
---

RSCの強力なところは、サーバーサイドならasync関数をコンポーネントとして使えることです。ということで、今回のハンズオンでもこれを試してみましょう。

## データを用意する

典型的な非同期処理のひとつはデータの読み込みです。ということで、今回は適当なJSONファイルを用意します。

```json:src/data/uhyo.json
{
  "name": "uhyo",
  "age": 29,
  "favoriteLanguages": [
    "TypeScript",
    "Rust"
  ]
}
```

## コンポーネントを用意して使ってみる

そして、それを読み込んで表示するコンポーネントを追加します。今回はdynamic importとともに`await`を使用しています。

```tsx:src/app/server/Uhyo.tsx
import type {} from "react/experimental";

export const Uhyo: React.FC = async () => {
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
};
```

ちなみに、`react/experimental`を読み込むのは、この本で採用している`@types/react`のバージョンにおいてasyncコンポーネントを利用するために必要です。

このコンポーネントを`App.tsx`から使います。

```diff tsx:src/app/server/App.tsx
 import { Client } from "./Client.js";
 import { Page } from "./Page.js";
+import { Uhyo } from "./Uhyo.js";
 
 export const App: React.FC = () => {
   return (
     <Page>
       <p>Hello, world!</p>
+       <Uhyo />
        <Client.Clock />
     </Page>
   );
 };
```

以上の変更でちゃんとasyncなコンポーネントが動作するはずです。サーバーを実行して`index.html`を出力→Viteで実行、という流れをとると以下のようにアプリが動作します。

![修正後のアプリのスクリーンショット](/images/rsc-without-nextjs/screenshot2.png)

ここで出力されているHTMLを見てみると、次のようになっています（整形済み）。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charSet="utf-8"/>
  </head>
  <body>
    <div id="app"><div><h1>React Server Components example</h1><p>Hello, world!</p><section><h2>Hello, I&#x27;m <!-- -->uhyo</h2><p>I am <!-- -->29<!-- --> years old.</p><p>My favorite languages are:</p><ul><li>TypeScript</li><li>Rust</li></ul></section><p></p></div></div>
    <script id="rsc-data" data-data="（略）"></script>
    <script type="module" src="src/client.tsx" async=""></script>
  </body>
</html>
```

asyncなコンポーネントがあっても問題なくSSRができていますね。ここまでの実装を反映したブランチは`step7`です。

ただし、これはまだRSCの真価を発揮していません。今回は静的な`index.html`を出力しているためストリーミングの恩恵を受けていないのです。いわゆるStreaming SSRとして、SSR結果そのもの（つまりHTML）をストリーミングすることもできるはずです。

そこで、次の章ではStreaming SSRの実装を目指しましょう。