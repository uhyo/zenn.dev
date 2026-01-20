---
title: "サーバーの無いReactフレームワークFUNSTACK Static"
emoji: "🏔️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

皆さんこんにちは。この記事では、筆者が最近開発した新しいReactフレームワークである**FUNSTACK Static**について紹介します。

https://github.com/uhyo/funstack-static

**[ドキュメント](https://uhyo.github.io/funstack-static/)**

## FUNSTACK Staticの概要

FUNSTACK Staticは、**React用のサーバーを立てる必要がなく、静的なファイルサーバーにデプロイできる古き良きSPA**を作ることに特化したフレームワークです。

それでいて、**React Server Components (RSC)** による最適化をしっかり効かせることができるのが特徴です。

[以前の記事](https://qiita.com/uhyo/items/cd5f8e6d55433286826e)で説明したように、RSCはサーバーというくらいだからサーバーをデプロイしないといけないと思われがちですが、実はそうではありません。RSCの恩恵を受けつつも、サーバーを立てずに静的ホスティングで動かせる方法があります。

Next.jsなど既存のフレームワークもそのようなモードをサポートしています。しかし、既存のフレームワークはサーバーを立てる機能を持っており、静的ホスティングのモードはおまけです。

また、サーバーを立てない場合は制限がかかると説明されることが多いでしょう。これは、サーバーが無い時点でRSCの恩恵を一部しか受けられないので当たり前ですが、制限がかかると言われてしまうと、やはり特殊なモードというか、あまり積極的には使わないものという印象を受けてしまいます。

対照的に、FUNSTACK Staticは**最初からサーバーを立てないことを前提**に設計されており、その制約の範囲でRSCの恩恵をしっかりと受けることを目指しています。

やっていることの大枠は一緒だとしても、サーバーを立てたくない人に対して、「サーバーを立てるとか言ってるよく分からないフレームワークの何か制限付きモードを使わなければいけない」という後ろ向きな認知ではなく、「**従来のSPAの方向性をそのままに、RSCで最適化可能**」という前向きな印象を持った選択肢を提供したいのです。

じゃあ中身は同じようなものかというと、そうでもありません。この記事でこれから説明しますが、FUNSTACK Staticは既存のフレームワークと全く同じというわけではありません。従来のSPAの開発体験に寄り添ったなかなかユニークな使い心地になりました。興味があればぜひ試してみてください。

## 使い方

細かいことは[ドキュメント](https://uhyo.github.io/funstack-static/)を参照しましょう（このドキュメントももちろんFUNSTACK Static製のSPAとして作られています。

FUNSTACK Staticは、一応フレームワークを名乗っていますが、独自のCLIなどは無く、実態は**Viteプラグイン**です。そのため、開発時に使うコマンドは`vite dev`や`vite build`など、Viteのコマンドそのままです。従来のSPA開発と一致した体験で好印象ですね。

FUNSTACK Staticを使うには、まずViteプロジェクトを作成し、そこにFUNSTACK Staticプラグインを追加します。後で説明しますが、2つのエントリーポイントの名前をオプションで提供してあげる必要があります。

```ts:vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import funstackStatic from '@funstack/static';

export default defineConfig({
  plugins: [
    funstackStatic({
      root: "./src/Root.tsx",
      app: "./src/App.tsx",
    }),
    react(),
  ],
});
```

現在のところ`root`と`app`の2つのエントリーポイントを固定で指定する必要がありますが、もしかしたら将来のアップデートでより柔軟になるかもしれません。

### Rootエントリーポイント

Rootエントリーポイントは、HTMLドキュメント全体を表すコンポーネントをエクスポートします。最も単純な形だと、こうなるでしょう。

```tsx:src/Root.tsx
import React from 'react';

export default function Root({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
        <meta charset="utf-8">
        <title>My FUNSTACK Static App</title>
      </head>
      <body>
        {children}
      </body>
    </html>
  );
}
```

このRootコンポーネントは次の特徴を持ちます。

- 実行時、`children`にはAppエントリーポイントの出力が入ります。
- Rootはサーバーコンポーネントであり、ファイルからの読み込みや非同期処理が可能です。
- クライアントコンポーネントを**使用できません**。

最後の点が特殊で、これはRootだけの独自の制約です。正確にはクライアントコンポーネントをインポートしてきて使用できますが、ハイドレーションされません。

その代わりに、Rootコンポーネントはビルド時に完全に静的なHTMLに変換されます。

### Appエントリーポイント

これがSPAのエントリーポイントとなるコンポーネントです。上述のRootコンポーネントの`children`としてレンダリングされます。

```tsx:src/App.tsx
import React from 'react';

export default function App() {
  return (
    <div>
      <h1>Welcome to my FUNSTACK Static App!</h1>
    </div>
  );
}
```

Appもやはりサーバーコンポーネントです。Appコンポーネント以下ではクライアントコンポーネントをインポートして使用することもできます。

```tsx
// 例: src/Counter.tsx
'use client';
import React, { useState } from 'react';

export const Counter = () => {
  const [count, setCount] = useState(0);
  return (
    <div>
      <p>Count: {count}</p>
      <button type="button" onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
};
```

```tsx:src/App.tsx
import React from 'react';
import { Counter } from './Counter';

export default function App() {
  return (
    <div>
      <h1>Welcome to my FUNSTACK Static App!</h1>
      <Counter />
    </div>
  );
}
```

つまり、サーバーコンポーネント/クライアントコンポーネントの区別があることに気を付ければ、従来のSPA開発とほぼ同じ感覚で`App`以下にアプリケーションを構築できるのです。

## RSCを使うメリット

本当に従来と同じ感覚でSPAを作りたいならサーバーコンポーネント/クライアントコンポーネントの分けなんて不要だよ、と思われるかもしれません。

しかし、メリットがあるから筆者はRSCをここまで推しているのです。SPAを作る観点から言えば、メリットは「ビルド時にコードを動かせる利便性」と「クライアントの負担軽減によるパフォーマンス向上」の2つです。

### ビルド時にコードを動かせる利便性

アプリの挙動を一部設定ファイルで管理したいなんてことは良くあります。従来は、`webpack.config.js`とか`vite.config.ts`に頑張って設定を詰め込みましたよね。

RSCであれば、**アプリケーションコードの一部という形で自由に設定ファイルを読み、処理できます**。

例えば、以下のように`config.json`を読み込んでアプリケーションの挙動を制御することができます。JSONファイルなどは直接importできるし、`fs`モジュールでファイルを読み込むこともできます。

```tsx:src/App.tsx
import React from 'react';
import config from './config.json';
import fs from 'node:fs/promises';

export default async function App() {
  const extraData = await fs.readFile('./extra-data.txt', 'utf-8');

  return (
    <div>
      <h1>{config.title}</h1>
      <p>{extraData}</p>
    </div>
  );
}
```

複雑なビルドステップを組まなくても、JavaScriptで普通に書いて自然にサーバーコンポーネントに組み込む。これが今どきのSPAのビルドのやり方です。

### クライアントの負担軽減によるパフォーマンス向上

RSCのメリットとして「**パフォーマンス向上**」が挙げられます。これは、RSCの性質である「サーバーコンポーネントのソースコードはクライアント向けのバンドルに含まれず、レンダリングの結果だけをクライアントが読み込む」という点に由来します。

まず、サーバーコンポーネントのソースコードがクライアントに送られないため、バンドルサイズが小さくなる可能性があります。ただし、代わりに「レンダリング結果のHTML」がクライアントに送られることになるため、コンポーネントの内容によってはあまりバンドルサイズに影響しない場合もあります。

次に、サーバーコンポーネントはクライアント側で実行されないため、クライアントのCPU負荷が軽減されます。Reactが「ただのHTML」をレンダリングするコストは、コンポーネントを実行してレンダリングするコストよりも安いです。

本当にクライアントコンポーネントである必要がある部分だけをクライアントコンポーネントにするというRSCの原則を実践することで、クライアントの負荷を低く抑えることができます。

## FUNSTACK Staticに無いもの

フレームワークとしてのFUNSTACK Staticの制約は、先に説明したものだけです。（将来的にエントリーポイントを増やせるようになるかもしれませんが）「rootとappの2つのエントリーポイントを指定する」というルール以外には、RSC自体のルールはもちろんありますが、フレームワークとしてのルールは特にありません。

FUNSTACK Staticには**ルーターも備わっていません**。ファイル名の規約とかは一切ありません。SPAとしてルーティングが必要であれば、React Routerとか、既存のSPA用ルーターライブラリを使うことになります。ちなみに、上述のドキュメントサイトでは[FUNSTACK Router](https://zenn.dev/uhyo/articles/funstack-router-poc)を使っています（宣伝）。

本当に、RSCによる最適化を効かせられるという点以外は、**トラディショナルなSPA開発と同じやり方**ができるのです。

また、当然ながら、FUNSTACK Staticはサーバーを立てる機能を持っていません（開発中に使用するViteの開発サーバーはもちろんあります）。よって、サーバーが必要なRSCの機能は使えません。具体的には、**Server Actions**はまったくサポートされていません。

あ、[ちょっと前に世間を騒がせたRSCの脆弱性](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)もFUNSTACK Staticにはありませんよ。サーバーが無いのですから。

## FUNSTACK Staticの仕組み

FUNSTACK Staticの特徴はそのビルド成果物にあります。

トラディショナルなSPAのビルド成果物は「エントリーポイントのHTMLファイル」と「バンドルされたJSファイル群」「CSSなどその他のアセット」です。

FUNSTACK Staticでも、これらの成果物は全て生成されます。それに加えて、**RSCペイロードがファイルに吐き出される**点が特徴です。RSCペイロードとは、サーバーコンポーネントのレンダリング結果を表すデータであり、クライアントがこれを読みこむことで、サーバーコンポーネントのレンダリング結果をブラウザ上で表示します（具体的には、RSCペイロードには「静的なHTML」と「クライアントコンポーネントの呼び出し」が含まれているので、それらをクライアント側のReactで処理します）。

| | クライアントでの動作 |
| --- | --- |
| 従来SPA | JavaScriptバンドル →（レンダリング）→ DOM |
| FUNSTACK Static | JavaScriptバンドル **+ RSCペイロード** →（レンダリング）→ DOM |

つまり、RSCによる最適化の恩恵とは、従来全て「JavaScriptバンドル」だったものを、一部（サーバーコンポーネント）を最適化してRSCペイロードというただのテキストとして分離し、JavaScriptバンドルを小さくしてクライアントの負荷を下げることに他なりません。

例えば、以下はドキュメントサイトの実際のビルド成果物の構成です。

```
public
├── FUNSTACK_Static_Hero_small.png
├── assets
│   ├── app-CrYJb6f5.js
│   ├── app-ovZqc1Hu.css
│   ├── app-pIc0I6Uv.css
│   ├── index-uNof5X04.js
│   ├── root-BLNqTaRx.css
│   └── rsc-ULMM1pWX.js
├── funstack__
│   ├── index.txt
│   └── rsc
│       └── fun:rsc-payload
│           ├── 29d259c01c528dda.txt
│           ├── 5b6e76f6acd28837.txt
│           ├── 6294448bd9a6bca0.txt
│           └── 6cd1c898f4fb6194.txt
└── index.html
```

`assets`以下が従来通りのクライアント用JavaScript/CSSアセット群です。それとは別に、`funstack__`以下にtxtファイルたちが出力されており、これがRSCペイロードです。

実際にFUNSTACK Static製のサイトを開いてデベロッパーツールでネットワークタブを見ると、これらのファイルが読み込まれているのが分かります。

そして、FUNSTACK Staticの**フレームワークとしての独自性**は、このRSCペイロードを吐き出す部分に集約されます。それ以外はおおよそ、先人たちの努力の成果である[@vite/plugin-rsc](https://github.com/vitejs/vite-plugin-react/tree/main/packages/plugin-rsc)を利用しています。

## `defer()` API

FUNSTACK Staticは、冒頭で紹介したViteプラグイン（`funstackStatic()`）のほかに、今のところ1つだけ、アプリケーション内で使えるAPIを提供しています。それが`defer()`です。

```ts
// サーバーコンポーネントから使えるAPIなので、@funstack/static/serverとして提供
import { defer } from '@funstack/static/server';
```

ここからは、`defer`が何なのか、なぜ必要なのかについて解説します。

### deferの概要

`defer`は一言で言えば、「**サーバーコンポーネント版の`lazy()`**」だと思っていただければよいです。`lazy()`はRSC以前からあるReactのAPIで、Suspenseと組み合わせて使うことで、クライアントコンポーネントの遅延読み込みを実現します。

```tsx
import React, { Suspense, lazy } from 'react';
const SomeComponent = lazy(() => import('./SomeComponent'));

export default function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      {/* SomeComponentは遅延読み込みされる */}
      <SomeComponent />
    </Suspense>
  );
}
```

特に、動的import（`import()`）と組み合わせて使うことで、バンドラによる**チャンク分割**が発生します。これにより最初のロード時に必要なJavaScriptバンドルのサイズを削減できるというのが、従来SPAで頻出のパフォーマンス最適化テクニックです。

同じ「チャンク分割」という考え方を、JavaScriptバンドルではなくRSCペイロードのほうに適用した結果生まれたのが`defer`です。

### RSCペイロードの分割の必要性

前述のとおり、RSCペイロードというのは、**サーバーコンポーネントのレンダリング結果を表すデータ**です。

そして、FUNSTACK Staticは、（rootは別として）Appというただ一つのエントリーポイントを持ちます。つまり、アプリケーションは要するに`<App />`のことなのです。

すると、`<App />`のレンダリング結果として1つのRSCペイロードが得られます。

つまり、`defer`がないと、「アプリ全体の内容を含んだ1つのRSCペイロード」ができるということです。

いくらRSCによる最適化があるといっても、これは`lazy`を全く使っていないのに近い状況ですから、さすがに無駄が大きいですね。例えば、次のような簡易的なルーティングを考えます。

```tsx
// src/Router.tsx
"use client";

interface RouterProps {
  homeContents: React.ReactNode;
  aboutContents: React.ReactNode;
}

export const Router: React.FC<RouterProps> = ({ homeContents, aboutContents }) => {
  const location = useLocation();

  switch (location.pathname) {
    case "/":
      return <div>{homeContents}</div>;
    case "/about":
      return <div>{aboutContents}</div>;
    default:
      return <div>Not Found</div>;
  }
};
```

```tsx:src/App.tsx
import { Router } from './Router';
import { HomePage } from './HomePage';
import { AboutPage } from './AboutPage';

export default function App() {
  return (
    <Router
      homeContents={<HomePage />}
      aboutContents={<AboutPage />}
    />
  );
}
```

このコードでは、なるべく多くをサーバーコンポーネントにするという基本的な考え方を活かして書かれたコードです。実際のルーティングは現在のURLが必要なので、クライアントコンポーネントである`Router`が担当します。

しかし、ページの中身はサーバーコンポーネントである`HomePage`や`AboutPage`が担当しています。クライアントコンポーネントのpropsとしてこのようにサーバーコンポーネント（のJSX Element）を渡すことができるので、こうすることで「**ルーティングのロジックはクライアントコンポーネントに任せつつ、ページの中身はサーバーコンポーネントで実装する**」という分担が可能になります。

このときに問題になるのが、**RSCペイロードのサイズ**です。`App`のレンダリング結果には`HomePage`と`AboutPage`の両方の内容が含まれることになります。

つまり、このSPAを開いたら、今どの画面を開いたとしても、「両方のページのレンダリング結果のHTML」をRSCペイロードとして全部読み込むことになるのです。これでは無駄にRSCペイロードが大きくなり、RSCの恩恵が薄れてしまいます。

### deferによるRSCペイロードの分割

実は、`defer`の型はこうなっています。

```ts
function defer(Component: React.FC<{}>): React.ReactNode;
```

上記のコードに`defer`を適用すると、こうなります。

```tsx:src/App.tsx
import { defer } from '@funstack/static/server';
import { Router } from './Router';
import { HomePage } from './HomePage';
import { AboutPage } from './AboutPage';

export default function App() {
  return (
    <Router
      homeContents={defer(HomePage)}
      aboutContents={defer(AboutPage)}
    />
  );
}
```

つまり、`<HomePage />`のように要素をレンダリングする代わりに`defer(HomePage)`とできるわけです。

`defer`を使った場合、**`defer`によりレンダリングされたサーバーコンポーネントは別々のRSCペイロードとして分離される**という効果があります。先ほど見せたビルド出力ファイルの一覧で`fun:rsc-payload`以下に複数のtxtファイルがあるのは、この`defer`による分割の結果です。

```
public
├── funstack__
│   ├── index.txt
│   └── rsc
│       └── fun:rsc-payload
│           ├── 29d259c01c528dda.txt ←
│           ├── 5b6e76f6acd28837.txt ←
│           ├── 6294448bd9a6bca0.txt ←
│           └── 6cd1c898f4fb6194.txt ←
```

そして、`defer`の結果は、以下のようなクライアントコンポーネントに置き換えられます。（以下は実際のコードではなく、ざっくりしたアイデアを示すものです）。

```tsx
"use client";
const DeferredComponent = ({ moduleId }) => {
  const rscPayload = getRSCPayloadStream(moduleId);
  return use(rscPayload)
};
```

`defer`は与えれたサーバーコンポーネントを別のRSCペイロードへとレンダリングし、そのペイロードのIDを発行します。`defer`の結果は`<DeferredComponent moduleId={...} />`というクライアントコンポーネントになります。

これが実際にクライアント側で実行された場合、この`DeferredComponent`がレンダリングされたとき（つまり`defer(...)`部分が実際にレンダリングされた場合）に初めて、対応するRSCペイロードがネットワークから取得されることになります。

これは確かに`lazy()`と似た動作ですね。違うのは、「クライアントコンポーネントのコード」を遅延読み込みするのではなく、「サーバーコンポーネントのレンダリング結果であるRSCペイロード」を遅延読み込みするという点です。

ちなみに、その性質上、`defer(...)`は**Suspenseでラップする必要があります**。ネットワークから読み込んだ結果をレンダリングするので必然ですね。

## まとめ

**FUNSTACK Static**は、従来のSPA開発の体験を大きく変えずに、RSCによる最適化を効かせられる新しいReactフレームワークです。

`defer`はそのコアであり、かつFUNSTACK Staticの独自の特徴です。プログラム内で動的に`defer`を呼び出してRSCペイロードを分割するという体験が従来`lazy()`を使ってきた方々にはなじみ深いはずです。

興味がある方は、使ってみたフィードバックや感想、応援をお寄せください。

### 補足

今回「サーバーを立てないRSC」をコンセプトとしたフレームワークを作って公開しましたが、実は筆者としては、別にReact用にサーバーを立てることに反対するわけではありません。むしろ、**パフォーマンスのために必要であればサーバーを立てて運用するのは有力な選択肢だ**と思っています。

それでも本文中に書いたように、サーバーを立てたくない人がそれでもRSCを活用するための**選択肢**があっても良いと思ったためにこれを作ったのです。

サーバーを立てないことを前提としたフレームワークを採用するいうことは、将来的にサーバーを立てる可能性を閉ざすことにもなります（頑張って移行すれば移行はできるかもしれませんが大変です）。サーバーを立てなかったことが、技術的負債となることも十分に考えられるのです。

筆者は昔、サーバーの無いSPAのパフォーマンスを良くするためにService Workerにまで手を出したことがあります。静的ファイルをキャッシュするだけみたいな良くあるものではなく、しっかりとロジックが載ったやつです。サーバーを立てなくていいことのメリットは小さくありませんが、頑張ってService Workerを実装してメンテナンスするのと、サーバーを立ててしまうのと、どちらが良かったのかはそんなに自明ではありません。まあ、当時はRSCなんてものは無かったのですが。

こういったことを踏まえた上で、FUNSTACK Staticを1つの選択肢としてご活用ください。

最後に水を差すような補足となりましたが、FUNSTACK Staticのコンセプトは我ながら面白いと思っており、満足しています。