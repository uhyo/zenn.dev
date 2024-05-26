---
title: "リソースの読み込みとメタデータ"
---

React 19では、`<head>`内へのレンダリングがサポートされました。従来、`<title>`、`<meta>`、`<link>`などhead内の要素をReactからコントロールすることはできず、`useEffect`のようなエスケープハッチでDOM操作したり、あるいはフレームワークに任せたりする必要がありました。

React 19ではコンポーネントの中に直接これらの要素をレンダリングできるようになりました。そうすると、Reactが自動的にhead内に要素を追加してくれます。

## メタデータをレンダリングする

例えば、`<title>`要素や`<meta>`要素、`<link>`要素をコンポーネント内に書くことができます。

```tsx:メタデータのレンダリング
const MyPage: React.FC = () => {
  return (
    <article>
      <title>My Page</title>
      <link rel="canonical" href="https://example.com/my-page" />
      <h1>My Page</h1>
      <p>My Page content</p>
    </article>
  );
};
```

### かぶったらどうなる？

こう聞いたときに気になるのは、例えば`title`を複数コンポーネントで使った場合どうなるのかです。これを実際に実験してみました。

答えは、特にワーニングなどは無く全部の`<title>`がheadに追加されます。ただし、順番が制御されているようです。コンポーネントツリー内の順序とは逆になり、一番後ろでレンダリングされた`<title>`がhead内で一番上になります。

```tsx
// Reactコード上の並び
<div>
  <title>A</title>
  <title>B</title>
  <title>C</title>
</div>

// 実際のhead内の並び
<title>C</title>
<title>B</title>
<title>A</title>
```

もしページの静的HTMLが元々`<title>`を持っていた場合、Reactが追加する`<title>`はその上に追加されます。

HTML仕様上`<title>`が2つ以上存在するのは明確な仕様違反なので、Reactは仕様違反のHTMLを生成することになります。

一方で、仕様に違反しているとはいえ複数の`<title>`要素が存在する場合にどれがページのタイトルとなるのかはHTML仕様で定義されており、最初のtitle要素が採用されます。

つまり、Reactはコンポーネントツリー内で一番後ろでレンダリングされた`<title>`要素が採用されるように制御していることになります。

## スタイルの読み込み

`<link>`要素をレンダリングできるということは、`<link rel="stylesheet">`を用いてスタイルシートを読み込むこともできるということです。

この場合、特定の条件を満たしていれば、**Suspense**と絡んだ特別な挙動をしてくれるようになります。

具体的には、指定されたスタイルシートの読み込みが完了するまで、その`<link>`がサスペンドするようになります。これにより、スタイルシートが読み込まれる前にコンテンツが表示されるという問題を回避できます。

この挙動をするために必要な条件とは、`precedence`属性をlink要素に与えることです。

```tsx:スタイルシートの読み込み
<div className="card">
  <link rel="stylesheet" href="card.css" precedence="default" />
  <h1 className="card-title">Card</h1>
  <p className="card-content">Card content</p>
</div>
```

この例では、`card.css`が読み込まれるまでこれらの要素はレンダリングされません。`.card`や`.card-title`、`.card-content`のスタイルが`card.css`に定義されている場合、それらがまだ読み込まれていないのにコンテンツが表示されるという問題を回避できます。

### precedence属性

ここで登場した`precedence`属性は、HTML仕様には存在しないReactの独自拡張です。これは、スタイルシートの適用順をReactに伝えて制御してもらうための仕組みです。

というのも、CSSでは、同じ詳細度のスタイルがある場合は後で書かれたスタイルが優先されます。そのため、意図した順番でスタイルシートを読み込むことが重要です。具体的に言えば、`<link>`要素の順番を制御できる必要があります。

その順番を`precedence`属性で指定することができます。上の例では`precedence="default"`としています。前述のサスペンドの挙動を得るためには`precedence`が必須なので、順序を意識しない場面でもとりあえず何らかの`precedence`を指定することが多いでしょう。

具体的に`precedence`属性に指定できる値は、実は**決まっていません**。どんな文字列でも構いません。`precedence`属性の扱いについては次のように[記述されています](https://react.dev/reference/react-dom/components/link)。

> React will infer that precedence values it discovers first are “lower” and precedence values it discovers later are “higher”.

つまり、Reactが最初に遭遇した`precedence`の値が最も優先度が低いものとなり、後に遭遇したものが高いものとなります。このため、うまく調整しないと意図しない順番になる可能性があります。

次の例では、Reactが先に`low`を発見し、次に`high`を発見しているため、`high`のスタイルシートが優先されます。

```tsx:precedence属性の例
const MyComponent: React.FC = () => {
  return (
    <div>
      <link rel="stylesheet" href="low.css" precedence="low" />
      <link rel="stylesheet" href="high.css" precedence="high" />
      <h1>My Component</h1>
    </div>
  );
};

// head要素内のレンダリング結果
<link rel="stylesheet" href="low.css" data-precedence="low">
<link rel="stylesheet" href="high.css" data-precedence="high">
```

しかし、もし別の場所で先に`high`が使われていたらどうでしょうか。

```tsx
<link rel="stylesheet" href="base.css" precedence="high" />
<MyComponent />
```

この場合、Reactは`high`を先に発見しているため、レンダリング結果は次のようになります。

```html
<link rel="stylesheet" href="base.css" data-precedence="high">
<link rel="stylesheet" href="high.css" data-precedence="high">
<link rel="stylesheet" href="low.css" data-precedence="low">
```

結構注意が必要ですね。本気でやるのであれば、ページの一番最初で使う`precedence`を全部列挙して、Reactに順番を認識させる必要があるでしょう。

幸い、`<link>`だけでなく`<style>`も`precedence`をサポートしています。なので、アプリの一番最初でこのようにしておくことで順番を定義できます。

```tsx
const App: React.FC = () => {
  // ...
  return (
    <>
      <style href="base-low" precedence="low"></style>
      <style href="base-default" precedence="default"></style>
      <style href="base-high" precedence="high"></style>
      {/* 他のコンポーネント */}
    </>
  )
}
```

なお、急に出てきた`<style>`の`href`もReactの独自拡張であり、インラインスタイルの重複排除のために使われます。これはTanStack QueryやSWCにおけるkeyと同じものだと考えると分かりやすいでしょう。`<style>`をレンダリングするコンポーネントを複数個同時にレンダリングした場合、それらが同じ`href`を持っている場合はReactが重複排除してくれます。

`<style>`をサスペンドに対応させるには`href`と`precedence`が必須なのでここでも書かれています。

本題の`precedence`の順序に関しては、このようにアプリの最初で全パターンを列挙することでReactに順番を認識させることができます。

:::message
`<style>`はインラインのスタイルシートですが、`@import`で外部スタイルシートを読み込んだりできるため`<link>`と同様にサスペンド機能を持っています。

他に`background-image`などから参照される画像の読み込みも待ってくれたら嬉しいですが、これは[仕様が明確ではない](https://github.com/w3c/csswg-drafts/issues/1088)ようなので依存しないほうがいいかもしれません。
:::

このような制御方法は何か馬鹿みたいだと思うかもしれません。しかし、CSSの`@layer`も大体似たようなものです。`@layer`は詳細度よりも強い制御力でスタイルの優先度を制御してくれますが、「先に出会ったものより後に出会ったもののほうが優先度が高い」という、Reactの`precedence`と同じような仕様になっています。一番上で次のように書くことも行われています。

```css
@layer low, default, high;
```

それと同じことと考えれば、Reactの`precedence`もまあそんなものだと思えるかもしれません。

## asyncスクリプト

これまで、`<link>`要素と`<style>`要素がSuspenseをサポートしていることをご紹介しました。他に、`<script async>`の場合も同様にサスペンドをサポートしています。

`<script>`の場合、`src`属性と`async`属性を両方指定することでSuspenseが有効化します。これにより、外部のスクリプトが読み込まれていることに依存するコンポーネントが簡単に書けます。

```tsx:スクリプトの読み込み
const MyComponent: React.FC = () => {
  return (
    <div>
      <script src="https://example.com/cute_gadget.js" async />
      <button type="button" onClick={() => CuteGadget.start()}>
        Start
      </button>
    </div>
  );
};
```

この場合、`cute_gadget.js`の読み込みが完了するとグローバル変数の`CuteGadget`が利用可能になるという想定です。Reactがサスペンドしてくれることによって、読み込みが完了するまでボタンが描画されないようにできます。

ちなみに、`<script>`のasync属性はHTML仕様に存在する概念であり、Reactの独自拡張ではありません。HTMLにおける`async`属性は、スクリプトの読み込みとHTMLのパースとを非同期に行うという意味があります。デフォルト（かつ`type="module"`ではない）の`<script>`要素は読み込み中HTMLのパースをブロッキングします。`defer`属性を指定した場合（および`type="module"`の場合）は、HTMLのパース完了後にスクリプトの実行が行われます。そして`async`属性を指定した場合は、HTMLのパースを止めずに読み込みを行い、読み込みが完了したらHTMLのパースが完了していなくてもそこで割り込んでスクリプトを実行するという意味になります。

なので、HTML仕様的な意味での`async`属性の存在と、ReactのSuspenseの挙動はそこまで必然的な関係がないような気もします。

## SSRとの関係

React 19でhead内の要素をコントロールできるようになったことは、SSRとの相性が良くなったとも言えます。これはReact 19におけるサーバーとの連携強化の一環と言えるでしょう。

前述のSuspense対応は、スタイルやスクリプトが読み込まれるより前に、それに依存するUIがレンダリングされるのを防ぐための仕組みです。SSRの場合、サスペンドという仕組みは必要なく、単純に必要なスタイルなどをhead要素内に出力すればよいだけです。このような厄介な最適化周りの対応をReactが全部やってくれるのはありがたいことです。例えばstyled-componentsなどのCSS in JSライブラリを使うページをSSRした経験がある方は、従来の大変さがちょっと分かるのではないでしょうか。

## プリロード系のAPI

最近のWebアプリにおいては、プリロード系の機能を用いて1ミリ秒でも早く必要なリソースの読み込みを行うようになっています。linkのrel属性で指定できるものだけでも、`preload`、`prefetch`、`prerender`、`preconnect`、`dns-prefetch`、`modulepreload`があり多岐にわたります。

Reactにおいても、「コンポーネントのレンダリング開始」と「DOMにレンダリングが反映される」タイミングにはラグがある場合も考えられます。その場合に、前者のタイミングで読み込みを開始できればちょっと時間を節約できます。

これに関連した4つの関数が`react-dom`に追加されていますので、使い方を[ブログ](https://react.dev/blog/2024/04/25/react-19)から引用します。

```tsx
import { prefetchDNS, preconnect, preload, preinit } from 'react-dom'
function MyComponent() {
  preinit('https://.../path/to/some/script.js', {as: 'script' }) // loads and executes this script eagerly
  preload('https://.../path/to/font.woff', { as: 'font' }) // preloads this font
  preload('https://.../path/to/stylesheet.css', { as: 'style' }) // preloads this stylesheet
  prefetchDNS('https://...') // when you may not actually request anything from this host
  preconnect('https://...') // when you will request something but aren't sure what
}
```

この中では`preinit`が特殊です。他は`<link rel>`と直接対応していましたが、`<link rel="preinit">`はありません。

`preinit`は、コメントに書かれている通り、`preload`に加えて読み込み（スクリプトの場合は実行）まで行ってしまうという意味になります。

これらのAPIは例によってSSRやサーバーコンポーネントにも対応しており、その場合もhead内に適切な要素が追加されます。

## まとめ

React 19でhead内の要素をコントロールできるようになりました。これにより、メタデータやスタイルシート、スクリプトの読み込みをReactのコンポーネント内で制御できるようになり、少し便利になりました。

特にSuspenseの対応は興味深いところです。今どき、ユーザーコードから完全に外部のスタイルシート等を読み込むということは少なそうですが、新たな形態のCSS in JSなど可能性を感じさせる機能ですね。