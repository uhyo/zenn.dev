---
title: "FUNSTACK Router: Navigation APIを用いたルーターライブラリ"
emoji: "🚗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

最近熱いWeb標準といえば、**Navigation API**ですね。これは、従来SPAを支えてきたHistory APIを置き換えるものです。Google Chrome 102、Firefox 147、Safari 26.2でサポートされており、（まだ安定版になっていないものも含めればですが）各ブラウザの最新版で利用可能なAPIです。

Navigation APIは、SPA向けのルーターライブラリ（例えばReact Router）が裏で利用するようなAPIです。しかし、そのような既存のルーターはこれまでHistory APIを使ってきたため、過去の遺産を背負いながらNavigation APIに対応しなければなりません。

そうなると、**History APIに引きずられない、Navigation APIを前提に作られたルーターライブラリ**がどのような様相になるのか気になりますよね。そこで、筆者が実装・公開したのが**FUNSTACK Router**です。

https://github.com/uhyo/funstack-router

## はじめに

FUNSTACK Routerは[ひとりNavigation API Advent Calendar 16日目](https://scrapbox.io/yamanoku/%E3%81%B2%E3%81%A8%E3%82%8ANavigation_API_Advent_Calendar_16%E6%97%A5%E7%9B%AE)で紹介していただきました。ありがとうございます。[ひとりNavigation API Advent Calendar](https://scrapbox.io/yamanoku/%E3%81%B2%E3%81%A8%E3%82%8ANavigation_API_Advent_Calendar)を読むとNavigation APIの概要や歴史を知ることができるので、ぜひそちらもご覧ください。

また、FUNSTACK Routerは、現在のところ、Navigation API時代のルーターライブラリの設計を示すための **PoC（Proof of Concept）** として位置付けています。実用的なルーターライブラリとして完成させるには、まだ多くの機能追加や改善が必要です。

## FUNSTACK Routerの基本的な使い方

[ドキュメント](https://uhyo.github.io/funstack-router/)（もちろんFUNSTACK Router製）も用意してあるので、詳しくはそちらをご覧ください。ここではドキュメントから引用したコード例を紹介します。

```tsx
import { Router, route } from "@funstack/router";

const routes = [
  route({
    path: "/",
    component: Layout,
    children: [
      route({ path: "/", component: Home }),
      route({ path: "/about", component: About }),
    ],
  }),
];

function App() {
  return <Router routes={routes} />;
}
```

このように、`routes`でルート一覧を定義してそれを`Router`でレンダリングするシンプルなAPIです。

## FUNSTACK Routerの特徴

では、Navigation APIを前提にしていることによって、FUNSTACK Routerはどのような特徴を持っているのでしょうか。いくつか挙げてみます。

### `<Link>`コンポーネントが不要

FUNSTACK Routerの特徴としてまず挙げておきたいのは、**`<Link>`コンポーネントが要らない**ことです。History APIとは異なり、Navigation APIでは、ただのリンク（a要素）をクリックしてナビゲーションが発生するのにフックする（navigationイベント）ことができます。したがって、既存のルーターライブラリのように`<Link>`コンポーネントを用意してそれを使わせる必要がありません。

```tsx
// 既存ルーターライブラリの場合
import { Link } from "react-router-dom";
<Link to="/about">About</Link>

// FUNSTACK Routerの場合
<a href="/about">About</a>
```

FUNSTACK Routerの`<Router>`コンポーネントの役割は、対応したコンポーネントをレンダリングすることもありますが、`navigate`イベントを監視してSPA的なナビゲーションに変換することでもあるのです。

ただし、これも難しくありません。`navigate`イベントのイベントオブジェクトに備わっている`intercept`メソッドを呼び出すだけで、ハードナビゲーションを中断させてSPA的なナビゲーションに変換できます。

```tsx
window.addEventListener("navigate", (event) => {
  // これだけでSPA的ナビゲーションに変換できる
  event.intercept();
});
```

### データ取得機構

FUNSTACK Routerは、ルートごとのデータ取得機能を備えています。ドキュメントから例を抜粋して引用します。

```tsx
function UserPosts(props: {
  data: Promise<Post[]>;
  params: { userId: string };
}) {
  return (
    <Suspense fallback={<div>Loading posts...</div>}>
      <UserPostsContent {...props} />
    </Suspense>
  );
}

const userPostsRoute = route({
  path: "/users/:userId/posts",
  component: UserPosts,
  loader: async ({ params }): Promise<Post[]> => {
    const response = await fetch(
      `https://jsonplaceholder.typicode.com/posts?userId=${params.userId}`
    );
    return response.json();
  },
});
```

コードの後半の`userPostsRoute`に注目すると、`loader`プロパティでデータ取得関数を指定しています。これにより、URLがこのルートにマッチして`UserPosts`コンポーネントがレンダリングされる際に、`loader`関数が呼び出され、その返り値が`data`プロパティとしてコンポーネントに渡されます。

面白い点は、**loaderは非同期関数に対応しているが、結果のPromiseはPromiseのままコンポーネントに渡される**ことです。非同期処理をどのように待つかはコンポーネント側に任せています。典型的には、`Suspense`と`use`を使って対応することになるでしょう。

そうなると、なぜ`loader`をFUNSTACK Routerの機能として用意するのかという疑問が湧くでしょう。`UserPosts`コンポーネント内で直接データ取得を行えば良さそうですよね。

そうせずに、FUNSTACK Router側でこの機能を用意している理由が2つあります。

理由の1つ目は、**ナビゲーションの完了を遅延させるため**です。ブラウザには、読み込みの最中であることを示すUIがあります。例えばGoogle Chromeでは、読み込み中のタブはfaviconの周りにローディングアニメーションが表示されます。また、読み込み中に中止ボタン（×ボタン）が表示されるものもありますね。

実は、Navigation APIでは、SPA的な遷移でも「読み込み中」状態を作ることができます。つまり、SPA内での「ページ遷移」の一環としてデータを読み込み中であることを、ブラウザのUIに反映させることができます。

FUNSTACK Routerでは、`loader`関数が解決されるまでナビゲーションの完了を遅延させ、その間ブラウザに「読み込み中」状態を示させることができます。以下は実際のFUNSTACK Routerの実装からの抜粋ですが、このように`event.intercept`に渡した`handler`が非同期関数となっており、これの完了を遅延させることでナビゲーションの完了が遅延される仕組みになっています。

```ts:NavigationAPIAdapter.ts
event.intercept({
  handler: async () => {
    // （省略）
    const results = executeLoaders(
      matched,
      currentEntry.id,
      request,
      event.signal,
    );

    // Delay navigation until async loaders complete
    await Promise.all(results.map((r) => r.data));
  },
});
```

理由の2つ目は、**ルーターがキャッシュ機構としてちょうどいいから**です。ReactのSuspenseでは、データ取得をコンポーネントの中ではなく、外にキャッシュを持つ必要があります。TanStack QueryなどのReact向けデータ取得ライブラリのコアの価値も実はキャッシュです。

特に、何をキーとしてキャッシュを持つのかがデータ取得ライブラリの設計を左右します。その点、Navigation APIにはキャッシュキーにとても適した仕組みがあります。履歴上で現在表示されているエントリーを表す`NavigationHistoryEntry`オブジェクトがあり、それらが固有の`id`を持っているのです。この`id`をキャッシュキーとして使えば、SPAの「ページ」に対してデータをキャッシュできます。これにより、ブラウザの戻る・進む操作で再度同じ「ページ」に戻った際にはキャッシュが働くため、再度データを取得する必要がなくなります。

これにより、ページ単位でデータ取得をするだけなら、他のデータ取得ライブラリに頼らずに、Suspense対応のステート管理を行うことができるのです。

History APIでも似たようなことは頑張ればできるのでしょうが、Navigation APIには標準でそのための仕組みが備わっているので、ルーターライブラリとしては実装がとても簡単になります。

## Navigation APIを完全にラップするのではなく共存できる

Navigation APIは、新しいだけあってよく設計されたAPIです。そのため、FUNSTACK Routerを使用している場合でも、ユーザーはNavigation APIを直接利用するという選択肢を持つことができます。

例えば、現在のところFUNSTACK Routerが利用しているNavigation APIのイベントは、`navigate`（ナビゲーションの発生にフックできる）と`currententrychange`（現在の履歴エントリーが変わったことを検知できる）だけです。前者は先ほど説明したとおりで、後者は「ページ遷移」を検知してRouterを再レンダリングするために使っています。

他のNavigation APIのイベントとしては、ナビゲーションに成功・失敗したことろ表す`navigatesuccess`や`navigateerror`というイベントがあります。FUNSTACK Routerはこれらのイベントを利用していないため、ユーザーはこれらのイベントにリスナーを登録して、ナビゲーション成功・失敗時に独自の処理を実行することができます。

このようなやり方ができるのは、Navigation API自身で履歴の管理を十分に行えるからです。History API時代に必要だったハック等は不要なので、ユーザーが直接Navigation APIを利用しても、FUNSTACK Routerと競合することは無いはずです。

これは言い換えると、FUNSTACK Routerは履歴管理を完全に独自に行うのではなく、あくまでNavigation APIとReactを結合するためのライブラリに過ぎない、ということでもあります。実際、FUNSTACK Routerのコア機能は独立した2つの機構に分かれています。一つは`navigate`イベントハンドリングする機構、もう一つは`useSyncExternalStore`を使って現在の履歴エントリをReactに同期させる機構です。これらはNavigation API内で相互作用しており、FUNSTACK Routerがそれらを直接結んでいるわけではありません。

ただし、今のところFUNSTACK Routerは`navigate`イベントをハンドリングするための`onNavigate` propを提供しています。これは、FUNSTACK Routerも`navigate`イベントをハンドリングしているため、ユーザーが直接`navigate`イベントを取り扱うと両者が競合したり、処理順が不明瞭になったりといった問題があるからです。

他にも、今のところプログラムからナビゲーションを発生させるための`useNavigate`フックを提供していますが、これも実はフックとして提供する意味が今のところ薄く、ユーザーが直接Navigation APIを叩いたとしても支障はありません。このAPIについては、AIにFUNSTACK Routerを実装してもらったらいつのまにかできていたのと、将来的に型安全性といった文脈で価値が出てくる可能性があるため、現時点では残しています。

## 将来の展望

Navigation APIができることはまだ色々あるので、FUNSTACK Routerも機能拡張の余地があります。例えば、従来History APIでは複雑な実装が必要だった「ページ遷移を防止する」ような実装も、Navigation APIでは`navigate`イベントをキャンセルすることで簡単に実現できます。これに対応する機能をFUNSTACK Routerに追加することが考えられます。

また、Navigation APIでは実はa要素によるページ遷移だけではなく、フォームの送信などあらゆるナビゲーションをフックできます。これらに対応する機能もFUNSTACK Routerに追加できるでしょう。

他にもView Transition APIとの連携など面白い機能が考えられます。

## まとめ

今回は、Navigation APIを前提にしたルーターライブラリである**FUNSTACK Router**を紹介しました。Navigation API時代のルーターライブラリのPoCとして、どのような設計になるのかを示すことができたと思います。

Navigation APIは近々ブラウザサポートも出揃いそうな期待のAPIなので、FUNSTACK Routerをもっとブラッシュアップして実用性を高めていきたいと考えています。興味のある方はぜひGitHubリポジトリを覗いてみてください。コードはそこまで多くありません。

使ってみた感想、issue、PRなどは大いに歓迎します。

https://github.com/uhyo/funstack-router