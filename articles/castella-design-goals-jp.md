---
title: "【日本語訳】CSS-in-JSライブラリ「Castella」のデザインゴール"
emoji: "🥯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "日本語訳"]
published: true
---

皆さんこんにちは。この記事ではReact向けCSS-in-JSライブラリ「[Castella](https://github.com/uhyo/castella)」を紹介します。Castellaのドキュメントには[Castella'a Design Goals](https://github.com/uhyo/castella/blob/master/docs/design-goals.md)という文書があり、Castellaがどのような思想で作られたCSS-in-JSライブラリなのか分かるようになっています。この記事ではこの文書の和訳をお届けします。

# Castellaのデザインゴール

Castellaは少しopinionated（訳注：特定のやり方を強制する）なCSS-in-JSライブラリです。この文書はなぜCastellaがこのようにデザインされているのか説明します。

## スタイルとマークアップを紐づける

[例](https://github.com/uhyo/castella/blob/master/README.md)を見ると分かるように、CastellaのAPIはCSS（スタイル）とHTML（マークアップ）を強く結びつけるような書き方を強制します。これは既存のライブラリ（あるいはCSS Modules）と対照的です。既存のライブラリはclassName指向であり、全てのCSSはひとつのクラス名を結果とし、そのクラス名は（訳注：マークアップ全体に対して適用されるのではなく）ただひとつの要素に適用されます。

スタイルは、たとえ目的が一つであっても複数の要素に分散しがちです。例えば、Flexboxレイアウトを使うためには親が`display: flex`を持つ必要があり、多くの場合は子の側も`flex: auto 1 0`のようなスタイルを持つ必要があります。複数の場所に散らばったスタイルはメンテナンスするのが難しいので、一つの目的に対するスタイルは一つの場所にまとめておきたいですね。

また、スタイルはマークアップに依存します。例えば、`display: flex`を持つ要素はその直接の子要素をレイアウトします。このことから、基となるマークアップを見ずにCSSを編集するのは勧められたことではありません。逆もまた然りです。

これらの理由から、一つの目的に対するCSSは、さらにそれに加えてHTMLも、一つの場所に集めたくなります。これはCastellaによって可能となります。

次の例は`TwoColumn`というコンポーネントを定義します。このコンポーネントの目的は2カラムレイアウトです。このコンポーネントの定義がこの目的のために必要な全てのスタイルとマークアップを含んでいるという点は注目に値します。

```tsx
export const TwoColumn = castella(
  css`
    display: flex;

    .aside {
      flex: 100px 0 0;
    }
    .main {
      flex: auto 1 0;
    }
  `,
  html`
    <div class="aside">${slot("aside")}</div>
    <div class="main">${slot("main")}</div>
  `
);

// 使い方
<TwoColumn
  aside={<p>Aside contents</p>}
  main={<p>Main contents</p>}
/>

// レンダリング結果
<castella-twocolumn-3c129a>
  #shadow-root (open)
    <style>
      :host {
        display: flex;
      }
      .aside {
        flex: 100px 0 0;
      }
      .main {
        flex: auto 1 0;
      }
    </style>
    <div class="aside"><slot name="aside"></slot></div>
    <div class="main"><slot name="main"></slot></div>
  <p slot="aside">Aside contents</p>
  <p slot="main">Main contents</p>
</castella-twocolumn-3c129a>
```

レンダリング結果は3つの要素から成っています。`<castella-twocolumn-3c129a>`は`display: flex`を持ち、`<div class="aside">`は`flex: 100px 0 0`を持ち、`<div class="main">`は`flex: auto 1 0`を持ちます。これらが、目的のレイアウトを実現するのに必要な全てです。外部から与えられたコンテンツは対応する`<slot>`の中に入れられます。

この例から分かるように、`TwoColumn`コンポーネントは一つの目的に必要なCSSとHTMLを過不足なく含んでいます。このようにして、Castellaはメンテナンスしやすいコンポーネントを定義する助けとなります。

## スタイルとロジックを分離する

我々のデザイン原則の一つは、スタイリングのためのコンポーネントとロジックのためのコンポーネントは分離すべきだということです。理想的には、全てのスタイルはスタイリングのためのコンポーネントの中に書かれ、ロジックのためのコンポーネントは一切のスタイルを認知しないべきです。また、ロジックのためのコンポーネントによって描画された要素はスタイリングのための役割を何も持つべきではありません。これらの要素はセマンティックな役割のみを持つべきです。

Castellaを使うことで、この方法論に従うことがほぼ強制されます。例えば、先ほどの`TwoColumn`をアプリで使ってみましょう。

```tsx
const App = () => {
  return <TwoColumn
    aside={
      <nav>
        <h2>Navigation</h2>
        <ul>
          <li><a href="/">Top</a></li>
          <li><a href="/diary">Diary</a></li>
        </ul>
      </nav>
    }
    main={
      <article>
        <h1>Welcome to our nice app!</h1>
        <p>Some contents</p>
      </article>
    }
  />;
}
```

ここで作った`App`コンポーネントはセマンティックな側面の役割に専従しています。スタイリングのために必要な`<div>`は全て`TwoColumn`コンポーネントの中に入っています。必要に応じて、`App`は`useState`などのロジックを含むこともできるでしょう。さらにスタイルが必要ならば、他のスタイリング用コンポーネントを作って使えばよいのです。

## スタイルを書く最高の体験

CSS-in-JSライブラリの重要な役割の一つは、スタイルのカプセル化です。大部分のCSS-in-JSライブラリがこれをユニークなクラス名を生成することで実現する一方で、CastellaはShadow DOMを利用します。（注：Castellaの`styled` APIはShadow DOMではなくクラス名に基づいています。）このことが最高のCSS開発体験をもたらします。

Castellaコンポーネントでは、（先ほどの`TwoColumns`の例のように）好きなクラス名を好きなように使うことができます。クラス名が他のコンポーネントと重複する可能性は考えなくて構いません。なぜなら、コンポーネントのスタイルとマークアップはShadow DOMの中に押し込められ、Shadow DOMの中のスタイルやマークアップは外の世界と干渉しないからです。Castellaのやり方はスタイルとマークアップを書く最も自然な書き方であり、しかもWeb標準に支えられています。

## 将来性

CastellaのAPIは、将来のWeb API（例えばに[Declarative Shadow DOM](https://web.dev/declarative-shadow-dom/)や[Constructable Stylesheets](https://developers.google.com/web/updates/2019/02/constructable-stylesheets)）とうまく適合するようにデザインされています。これらのAPIが利用可能になったときに、Castellaのユーザーは最小限の努力で新しいAPIの恩恵を受けることができます。

## オプトアウトしやすい

Castellaは、[Babel Macro](https://github.com/kentcdodds/babel-plugin-macros)によるプログラム変換によって作られています。これは、Castellaコンポーネントは完全に静的解析可能であるということです。もしCastellaをやめて他のCSS-in-JSに移りたくなった場合でも、Castellaから他の手段への変換を実装することで、最小限の努力でCastellaをやめることができます。

# 種明かし

ということで、[Castella'a Design Goals](https://github.com/uhyo/castella/blob/master/docs/design-goals.md)の和訳をお届けしました。

こうして紹介するとCastellaは海外で人気のイケてるCSS-in-JSライブラリに見えますが、実はそんなことはなくCastellaのGitHubスター数はたった16です（記事執筆時点）。というか、Castellaを作ったのはこの記事を書いた人だし自分で書いた英文を自分で日本語に訳しているだけです。この記事はCastellaを宣伝したかっただけです。

しかし、自画自賛ですがCastellaはCSS-in-JSライブラリとして優れたインターフェースを備えていると自分としては信じて疑わないところです。いいなと思ったらぜひ使ってみてください。