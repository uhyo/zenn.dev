---
title: "React 19.3 browser() APIの使いみち～FUNSTACK Routerの場合～"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

皆さんこんにちは。最近、React公式ドキュメントに[browser() API](https://react.dev/reference/react-dom/browser)の記事が追加されました。この記事公開時点ではまだ正式版のReactでは使用できませんが、次バージョンのReact 19.3で利用可能になると見込まれています。

この記事では、筆者が開発しているOSS [FUNSTACK Router](https://router.funstack.work/)にさっそくこの `browser()` APIを導入してみたので、どんなユースケースで使ったのかをご紹介します。

## `browser()` APIとは

`browser()` APIは、あるコンポーネントが**SSRではレンダリングできず、ブラウザ上でのみレンダリングできる**ことを示すAPIです。コンポーネント内で、`use()`と組み合わせて使用します。上記公式ドキュメントからコード例を引用します。

```tsx
import { use } from 'react';
import { browser } from 'react-dom';

function BrowserOnly() {
  use(browser('This component requires browser APIs.'));
  return <BrowserContent />;
}
```

この例では、`<BrowserOnly />` はSSRではレンダリングできなくなります。また、`use()`はコンポーネント内に加えてカスタムフック内でも使えるし条件分岐の中でも使えるというルールのため、次のような書き方も可能です（公式ドキュメントより引用）。

```tsx
function useBrowserQuery(query, options) {
  if (options.initialData === undefined) {
    use(browser('useBrowserQuery: No initial data was provided.'));
  }

  return useQuery(query, options);
}

function ProductDetails({ productId, initialData }) {
  const product = useBrowserQuery(`/api/products/${productId}`, {
    initialData,
  });

  return <h1>{product.name}</h1>;
}
```

この例では、 `<ProductDetails />`をSSRでレンダリングする場合は`initialData`が必要という制約を表現しています。さもなければ、このコンポーネントはSSRではレンダリングできず、ブラウザ上でのみレンダリングされることになります。

### SSRでレンダリングできないとはどういうことか

SSRは、コンポーネントをWebサーバー（あるいはビルド時など）でレンダリングしてHTMLをあらかじめ生成する仕組みです。それを初期HTMLとしてレスポンスを返すことで、ユーザーはページを開いた瞬間からコンテンツを目にすることができ、UXの向上やSEOの改善につながります。ブラウザ上でもReactのレンダリングが実行され、ハイドレーションが行われます。

では、`use(browser())`を使っているコンポーネントをSSRでレンダリングしようとした場合、どうなるでしょうか。この場合、**直近のSuspenseバウンダリがフォールバック化する**という挙動になります。つまり、初期HTMLにはそのコンポーネントの内容は含まれず、SuspenseのフォールバックUIが代わりに含まれることになります。

クライアント上でハイドレーションが行われると、今度はそのコンポーネントがブラウザ上でレンダリングに成功します。そうなると、SuspenseのフォールバックUIは消え、コンポーネントの内容が表示されることになります。

つまり、メンタルモデルとしては、SSRでは疑似的なサスペンドが発生したような挙動になると考えるとわかりやすいです。SSRでレンダリングできないコンポーネントは、実際にブラウザ側でレンダリングが行われるまで**待つ**必要があるコンポーネントだということですね。

ちなみに、この挙動は、SSR中にエラーが発生した場合の挙動を応用したものです。従来から、ReactはSSR中に特定のコンポーネントでエラーが発生した（例外が投げられた）場合、その周りのSuspenseバウンダリをフォールバック化させ、クライアント側で再度レンダリングを試みるという挙動をしていました。

つまり、従来からコンポーネント内でエラーをthrowすれば、「SSRではSuspenseのフォールバックUIを表示し、クライアント側で再レンダリングする」という挙動が実現できていました。しかし、これはあくまでエラー扱いのため、Reactのエラーハンドリングに乗ってしまい、エラーログに繋がるといった問題がありました。

そこで、Reactが公式に、このユースケースのための「エラー扱いにならない特別な例外」として用意したのが`browser()` APIであると捉えられます。

:::details 余談: RSCとの関係
今回の話題は、RSC (React Server Components)とは特に関係ないということに注意してください。RSCを使っていないアプリケーションでも、SSRをしているのであれば`browser()` APIの利用価値があります。

RSCでは「サーバーコンポーネント」と「クライアントコンポーネント」の2種類の環境にコンポーネントが分類されますが、`browser()` APIを使用できるのは「クライアントコンポーネント」の側です。

RSCの文脈では、SSRは「基本的にはブラウザ側で描画するためのクライアントコンポーネントを、サーバー側でもレンダリングすること」を指します。そのため、`browser()`はクライアントコンポーネントで使うのです。
:::

## FUNSTACK Routerでのユースケース

FUNSTACK RouterはReact向けのルーターライブラリです。つまり、React RouterやTanStack Routerといったライブラリの仲間です。

FUNSTACK Routerでは、SSRをサポートしています。その中でも、今回関係するのは**pathless SSR**というモードです。これは、**URL（パス）が不明な状態でSSRする**ことです。

ルーターライブラリの主な役割は、URL（パス）に応じてどのコンポーネントをレンダリングするかを決定することです。つまり、完全にSSRをするためには、SSR時点でURL（パス）がわかっている必要があります。例えば、サーバー側でリクエスト時にSSRする場合はリクエストからURLを取得します。

しかし、FUNSTACK Routerでは、URLが不明なままSSRを行うことができます。これは、**1つの静的HTMLをSPAとして配信したいけど、ビルド時に最適化目的でSSRできるところはしておきたい**というユースケースを想定した機能です。パスありSSRをビルド時に行う場合、静的サイトジェネレータのように全てのパスを個別に事前レンダリングしておく必要があります。そこまでせず、1つのHTMLだけ生成しておいてSPAとして配信したいというユースケースです。

FUNSTACK Routerには、最近のルーターライブラリよろしく、**ネストしたルート定義**や**レイアウトルート**の機能があります。たとえば、このようなルート定義をしたとします。

```tsx
const routes = [
  route({
    component: AppShell,
    children: [
      route({ path: "/", component: HomePage }),
      route({ path: "/about", component: AboutPage }),
    ],
  }),
];
```

直接の`routes`の要素は1つだけで、`AppShell`がルートコンポーネントとして定義されています。このルートは`path`定義がないため、**pathlessルート**と呼ばれます。このようなルートは常にマッチします。`AppShell`は内部で`<Outlet />`を使って子ルートのコンポーネントをレンダリングします。

```tsx
const AppShell = () => {
  return (
    <div>
      <Header />
      <Outlet /> {/* ←子ルートのコンポーネントがここにレンダリングされる */}
      <Footer />
    </div>
  );
};
```

この例だと、URLが`/`の場合は`<Outlet />`の部分に`<HomePage />`がレンダリングされ、`/about`の場合は`<AboutPage />`がレンダリングされます。

### Pathless SSRでのOutletの挙動と従来の問題点

ここからが本題です。Pathless SSRモードでは、URLが不明な状態で`routes`をレンダリングすることになります。

この場合、pathlessルートはSSR可能であり、`AppShell`はSSRでレンダリングできます。

しかし、`<Outlet />`に何を入れればいいか分かりません。この中身はpathless SSR時には決定できないのです。そのため、従来のFUNSTACK Routerでは、pathless SSR時の`<Outlet />`は**空のままレンダリングされる** （`null`がレンダリングされる）という挙動になっていました。

すると、SSR時の初期HTMLではシェルだけがレンダリングされており、ページ本体部分（`<Outlet />`の部分）は空のままです。クライアント側でハイドレーションが行われると、URLが判明するので、`<Outlet />`の中身がレンダリングされます。

一見良さそうですが、ひとつ問題がありました。それが**ハイドレーションミスマッチ**です。SSR時とブラウザ上でのレンダリングで`<Outlet />`の結果が異なると、レンダリングされるDOMが異なるということになるので、ハイドレーション時にエラーが発生してしまいます。

### `browser()` APIの導入

そこで、今回のReact 19.3の`browser()` APIを導入することで、この問題を解決しました。最新版のFUNSTACK Routerでこの挙動にオプトインすると（`<Router features={{ pathlessSSROutletDeferral: true }}>`）、上記の状況では`<Outlet />`が`use(browser())`するようになります。

つまり、SSR時に`<Outlet />`の中身をURL無しで決定できない場合は、`<Outlet />`がサスペンドし、SuspenseのフォールバックUIが表示されることになります。クライアント側でハイドレーションが行われると、URLが判明するので、`<Outlet />`の中身がレンダリングされます。

SSR時にSuspenseのフォールバックUIに落ちたところがクライアント側でレンダリング成功したとしてもハイドレーションミスマッチではないので、これにより従来挙動の問題を防ぐことができます。

このため、ルート定義側では、必要に応じて`<Outlet />`をSuspenseで囲む必要があります。

```tsx
const AppShell = () => {
  return (
    <div>
      <Header />
      <Suspense fallback={<Loading />}>
        <Outlet /> {/* ←子ルートのコンポーネントがここにレンダリングされる */}
      </Suspense>
      <Footer />
    </div>
  );
};
```

ちなみに、FUNSTACK Routerではもともとルート定義にデータローディングを紐づける機能があったので、`<Outlet />`をSuspenseで囲むことは特別新しい習慣というわけではありません。

## まとめ

この記事では、React 19.3で導入される予定の`browser()` APIを使って、FUNSTACK Routerのpathless SSRモードでの`<Outlet />`の挙動を改善した例をご紹介しました。

この`browser()` APIは、従来はエラー時のリカバリの挙動だった「エラー時にフォールバックUIに落ちる」という挙動を応用して、「SSR時にはまだレンダリングできない（疑似的なサスペンド）」として既存のSuspenseの概念に乗せたのが面白い設計だと思います。

FUNSTACK Router側でも、「`<Outlet />`をまだレンダリングできない、という事象が発生する要因がSSR時には1個増えた」という形で、自然に`browser()`の挙動を取り入れることができました。

余談ですが、FUNSTACK Routerは、いわゆるAsync ReactのようなモダンなReactの設計を積極的に採り入れることを目指しています。今回の`browser()`もその好例となり、筆者としては満足しています。