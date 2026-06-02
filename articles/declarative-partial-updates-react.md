---
title: "Declarative Partial UpdatesをストリーミングSSRに使う"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["web"]
published: true
---

**Declarative Partial Updates （宣言的部分更新）** は、Googleによって提案されているWeb標準（候補）です。詳しくは以下のページをご覧ください。

https://developer.chrome.com/blog/declarative-partial-updates

GoogleのWeb標準というと、他のブラウザベンダーのサポートを得られないまま突っ走ることも多い印象ですが、このDeclarative Partial Updates（略してDPU）は、**FirefoxとWebkitからも肯定的な反応が得られている**ようです。その意味で期待が持てるWeb標準です。

DPUは関連する新しいJavaScript APIなども含んでいますが、今回は例によって、**JavaScript無しでHTMLを後から更新できる**点に注目します。

## Declarative Partial Updatesの機能

Webサイトを表示するときは、HTMLがWebサーバーからストリーミングされてきます。Webブラウザは、ストリーミングが完了する前でも、現在まで受け取った部分を順次画面に表示する機能を備えています。

この機能を活用するにあたって、HTMLは文字列として前から後にしか送られてこないので、必然的にページの前から順番に表示される体験になっていました。

DPUはこの弱点を克服し、**HTMLの任意の部分を後から更新できる**ようにする機能です。

以下のHTMLを考えます（以下、コード例は上記のGoogleの記事から引用）。

```html
<div>
  <?marker name="placeholder">
</div>

...

<template for="placeholder">
  Here is some <em>HTML content</em>!
</template>
```

この例では、`<div>`の中に`<?marker name="placeholder">`というマーカーが置かれています。これは、**この場所を後から更新するための目印**です。この段階では、マーカーは何も表示しません。`<div>`は空のままです。

その後、`<template for="placeholder">`で、マーカーの場所を更新するためのHTMLコンテンツがストリーミングされてきます。これがブラウザに到達すると、`<template>`の中身がマーカーの位置に流し込まれることになります。つまり、最終的には以下のように表示されることになります。

```html
<!-- 最終的な表示 -->
<div>
  Here is some <em>HTML content</em>!
</div>
```

これにより、「一度ストリーミングし終わった`<div>`の中身を後から別途ストリーミングする」ということができます。

また、後から中身が送られてくるまでの間に表示するプレースホルダを指定することもできます。

```html
<div>
  <?start name="another-placeholder">
  Loading…
  <?end>
</div>

...

<template for="another-placeholder">
  Here is some <em>HTML content</em>!
</template>
```

マーカーが`<?start>`と`<?end>`になりました。これにより、マーカーの位置に「Loading…」というプレースホルダが表示されるようになります。後から`<template>`の内容が到達すると、プレースホルダは置き換えられます。

## DPUのユースケース

この説明を見たとき、どう思われましたか。「何に使うんだろう」と思われた方も多いかもしれません。

色々な使い道があるかもしれませんが、1つ明確な答えがあります。**Reactが使います**[^note_react]。Reactでなくても、SSR機能を備えたUIライブラリなら活用できるでしょう。

[^note_react]: 現実には、Reactは後方互換性を重視するので、DPUをすぐに活用するのは難しいかもしれません。ここで言いたいのは、Reactは明確なユースケースの1つであるということです。

というのも、筆者はDPUを初めて知ったとき「**ReactのストリーミングSSRそのまんまやん！！**」と思いました。

ReactのストリーミングSSRは、サーバー側でReactコンポーネントをレンダリングしてHTMLを生成し、それをストリーミングしてクライアントに送る機能です。ポイントは**Suspenseのサポート**です。このようなアプリをストリーミングSSRすることを考えましょう。

```jsx
<h1>My App</h1>
<Suspense fallback={<div>Loading...</div>}>
  <MyComponent />
</Suspense>
```

Reactアプリケーションとして見ると、コードの基本的な動きは以下のとおりです。

1. 最初のレンダリングでは、Suspenseのフォールバックが表示される。
2. `MyComponent`のレンダリングが完了すると、Suspenseのフォールバックが置き換えられる。

この動きをHTMLのストリーミングでも実現するのがReactのストリーミングSSRです。Reactは、まず`<h1>My App</h1><div>Loading...</div>`をストリーミングしてクライアントに送ります。クライアントはこれを受け取って表示します。その後、サーバー側で`MyComponent`のレンダリングが完了すると、Reactは`<div>Loading...</div>`を`MyComponent`のHTMLに置き換えるためのHTMLをストリーミングします。

やっていることがまんまDPUですよね。Reactは、今のWeb標準でこの動きをするために、2のタイミングで **インライン&lt;script&gt;** を送ってきて、DOM操作で無理やりDPUのような動きを実現しています。

DPUがあれば、Reactはこの動きをインラインスクリプトに頼らずに、HTMLだけで実現できるようになります。

## 実際にやってみた

https://github.com/uhyo/jsx-partial-updates

ということで、実際にやってみました。このリポジトリでは、ミニマルなJSXランタイムを実装しました。このランタイムはストリーミングSSRだけを実装しています。

```jsx
const Page = () => <div><h1>Hello!</h1></div>;

const stream = renderToStream(<Page />); // "<div><h1>Hello!</h1></div>"をストリーミングする
```

このランタイムにはReactのSuspenseを模した、特別な`<Sasupensu>`コンポーネントが実装されています。例えばこのような使い方ができます。

```tsx
<Card title="Activity feed">
  <Sasupensu fallback={<Spinner label="loading feed…" />}>
    <Feed />
  </Sasupensu>
</Card>
```

:::details デモページのコンテンツ全文

```tsx
async function Profile() {
  await sleep(800);
  return (
    <div class="card-body">
      <p class="name">Ada Lovelace</p>
      <p class="muted">Joined 1843 · 4 followers</p>
    </div>
  );
}

async function Stats() {
  await sleep(400);
  return (
    <ul class="stats">
      <li><strong>128</strong> commits</li>
      <li><strong>12</strong> open PRs</li>
      <li><strong>3</strong> reviews</li>
    </ul>
  );
}

async function Comments() {
  // Resolves *after* its parent <Feed> — demonstrates a nested boundary, which
  // can only be patched once its parent's template has been applied.
  await sleep(1200);
  return (
    <ul class="comments">
      <li>“Streaming makes this feel instant.”</li>
      <li>“No client JS required for the swap 🤯”</li>
    </ul>
  );
}

async function Feed() {
  await sleep(1500);
  return (
    <div class="card-body">
      <p>Latest activity loaded at {new Date().toLocaleTimeString()}.</p>
      <h3>Comments</h3>
      <Sasupensu fallback={<Spinner label="loading comments…" />}>
        <Comments />
      </Sasupensu>
    </div>
  );
}

function Spinner({ label }: { label: string }) {
  return (
    <p class="spinner">
      <span class="dot" aria-hidden="true" /> {label}
    </p>
  );
}

function Card({ title, children }: { title: string; children?: JSXNode }) {
  return (
    <section class="card">
      <h2>{title}</h2>
      {children}
    </section>
  );
}

/* -------------------------------------------------------------------------- */
/* The page                                                                   */
/* -------------------------------------------------------------------------- */

function Page({ polyfill }: { polyfill: boolean }) {
  return (
    <html lang="en">
      <head>
        <meta charset="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <title>jsx-partial-updates · streaming server demo</title>
        <style dangerouslySetInnerHTML={{ __html: STYLES }} />
        {/*
          Demo-only polyfill so the swaps are visible in any browser. It is
          opt-in: add `?polyfill` to the URL to enable it. Without it, the
          swaps only happen in a browser with native Declarative Partial
          Updates support (Chrome 148+ behind a flag).
        */}
        {polyfill ? (
          <script dangerouslySetInnerHTML={{ __html: PARTIAL_UPDATES_POLYFILL }} />
        ) : null}
      </head>
      <body>
        <header>
          <h1>Streaming JSX with &lt;Sasupensu&gt;</h1>
          <p class="muted">
            This whole page is one streamed HTTP response. The header and the
            three spinners arrived first; each card pops in on its own as its
            data resolves — fastest first.
          </p>
        </header>

        <main class="grid">
          {/* Declared first, but the slowest — its template streams last. */}
          <Card title="Activity feed">
            <Sasupensu fallback={<Spinner label="loading feed…" />}>
              <Feed />
            </Sasupensu>
          </Card>

          <Card title="Profile">
            <Sasupensu fallback={<Spinner label="loading profile…" />}>
              <Profile />
            </Sasupensu>
          </Card>

          <Card title="Stats">
            <Sasupensu fallback={<Spinner label="loading stats…" />}>
              <Stats />
            </Sasupensu>
          </Card>
        </main>

        <footer class="muted">
          Rendered by jsx-partial-updates · the footer streamed before any card
          finished loading.
        </footer>
      </body>
    </html>
  );
}
```

:::

このJSXランタイムは、Sasupensuバウンダリをレンダリングする際に、DPUのマーカーとフォールバックを出力します。

```html
<section class="card">
  <h2>Activity feed</h2>
  <?start name="S:0">
    <p class="spinner"><span class="dot" aria-hidden="true"></span> loading feed…</p>
  <?end>
</section>
```

その裏で`<Feed />`（時間がかかる非同期コンポ―ネント）のレンダリングが進んでおり、それが完了したら以下のようなHTMLがストリーミングされてきます。

```html
<template for="S:0">
  <div class="card-body">
    <p>Latest activity loaded at 10:49:31 PM.</p>
    <h3>Comments</h3>
    <?start name="S:3">
    <p class="spinner"><span class="dot" aria-hidden="true"></span> loading comments…</p>
    <?end>
  </div>
</template>
```

これにより、DPUが働き、`loading feed…`の部分が`<Feed />`の内容で置き換えられることになります。

余談ですが、Sasupensuはネストさせることもでき、その場合は送られてきたtemplateの中にさらにマーカーが入ることになります。上の例では、`<Feed />`の中にさらに`<Comments />`という非同期コンポーネントが入っているため、`S:3`というマーカーも入っています。

興味がある方はこのデモをぜひ自分で実行してみてください。Google Chrome 148（記事執筆時点の最新安定版）にフラグ付きで実装されています。デモの実行手順は以下をご覧ください。

https://github.com/uhyo/jsx-partial-updates/blob/master/README.ja.md

## まとめ

この記事では、Declarative Partial Updates (DPU) がReactのようなUIライブラリによるストリーミングSSRというユースケースにとても良く合致していることを説明しました。

また、実際にミニマルなJSXランタイムでストリーミングSSRを実装することで、DPUによるストリーミングSSRが実現可能なアイデアであることを示しました。

近年、JavaScriptに頼らずHTMLで何でもできるようにする動きが強化されており、DPUもその一部という見方もできます。

それはときにReactのようなUIライブラリと対立するものとして語られることもありますが、実際にはUIライブラリはHTMLやDOMの新機能から恩恵を受け、進化する立場なのです。