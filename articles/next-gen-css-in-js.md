---
title: "次の時代のCSS in JSはWeb Componentsを従える"
emoji: "🚻"
type: "tech"
topics: ["javascript", "react", "WebComponents"]
published: true
---

**CSS in JS**はJavaScriptのコード中にCSSを書くプログラミング手法で、[styled-components](https://styled-components.com/)などがメジャーどころです。現代的な開発では、ReactやVueといったフロントエンド用のViewライブラリを使う際にCSS in JSのお世話になることがよくあります。

この記事では、**次の時代のCSS in JSはWeb Componentsベースのものになるのではないか**という話をします。

## Web Componentsの復習

Web ComponentsはいくつかのWeb標準の総称であり、特に重要なのは**Custom Elements**や**Shadow DOM**です。いずれも、V1と呼ばれる仕様は全てのモダンブラウザでサポートされています（Safariが一歩遅れていて少し心配ですが）。

- https://caniuse.com/custom-elementsv1
- https://caniuse.com/shadowdomv1

Custon Elementsは`<x-foo>...</x-foo>`のような独自の要素名を登録して、要素が作られた時の処理をどうこうしたりできる機能です。Shadow DOMはもっと重要なので、まだ知らない方向けに少し詳しく解説します。

### Shadow DOM

Shadow DOMは要素の「内部実装」のようなものを定義できる機能です。例えば、次のdiv要素は中身に`p`があるだけの単純な構造をしています。

```html
<div id="area" style="border: 1px solid #666666">
  <p>Hello, world!</p>
</div>
```

![div要素の表示](https://storage.googleapis.com/zenn-user-upload/k6lwjkeoiszd4otyuhvi12lbazxm)

しかし、Shadow DOMをdiv要素にアタッチすることで、div要素に複雑な構造を与えることができます。

```js
const div = document.getElementById('area');
// divにShadow Rootをアタッチする
const shadowRoot = div.attachShadow({ mode: 'open' });
// Shadow Rootの中身を書く
shadowRoot.innerHTML = `
<style>
  p {
    border: 1px solid red;
  }
</style>
<p>↓↓↓↓↓</p>
<slot></slot>
<p>↑↑↑↑↑</p>
`;
```

こうすると、div要素は次のように表示されます。

![shadowRoot追加後のdiv要素の表示](https://storage.googleapis.com/zenn-user-upload/c5qg9xlj7ij5res9563rqbwaga76)

つまり、div要素にShadow DOMをアタッチしたことで、それがdiv要素の中身として表示されるようになります。また、Shadow DOM内で`<slot></slot>`とありますが、この部分にdiv要素の子（今回は`<p>Hello, world!</p>`）が入ります。よって、div要素はあたかも次の中身を持っているかのような表示になります。

```html
<div id="area" style="border: 1px solid #666666">
  <style>
    p {
      border: 1px solid red;
    }
  </style>
  <p>↓↓↓↓↓</p>
  <p>Hello, world!</p>
  <p>↑↑↑↑↑</p>
</div>
```

ただし、ここでShadow DOMの中に`style`要素が書かれていることは注目に値します。これは`p`に対するスタイル指定を持っていますが、**Shadow DOM内に書かれたスタイルが影響力を持つのは同じShadow DOM内の要素に対してだけです**。よって、同じShadow DOMに由来する`<P>↓↓↓↓↓</p>`と`<p>↑↑↑↑↑</p>`には`border: 1px solid red;`が適用されますが、`<p>Hello, world!</p>`はShadow DOMに由来しないので`border: 1px solid red;`が適用されません。

逆に、Shadow DOM外に次のようなスタイルを書いたとしても、Shadow DOM内には影響しません。

```css
/* Shadow DOM外に書く */
p {
  color: blue;
  font-size: 2em;
}
```

![スタイル追加後の表示](https://storage.googleapis.com/zenn-user-upload/oohqw2w2wgutd7ivrb5yiqz6jkrz)

このように、Shadow DOMの中と外はスタイルが隔絶しています。CSS in JSを考えるにあたってはこれが一番重要です。

## ViewライブラリとWeb Components

Web Componentsは、名前に「Components」とあることから分かるように、Web標準にコンポーネントの概念を持ち込む試みです。一方、我々はReactやVueなどのViewライブラリを通してコンポーネントという概念に慣れ親しんでいます。そのため、Web Componentsに対しては「ライブラリを使わなくてもコンポーネントを作れる」「ViewライブラリからWeb標準への以降」という視点がよく見られます。

しかし、筆者の考えは違います。そもそもViewライブラリはDOMというWeb標準に上に作られたライブラリであり、エコシステムです。であるならば、Web標準の進化はViewライブラリの衰退ではなく、むしろ進化に繋がるでしょう。ReactなどのViewライブラリのエコシステムも、Web Componentsを土台にしたものへと移行すると期待されます。これがこの記事の予言です。

そして、Web Componentsの恩恵を比較的受けやすいのが**CSS in JS**であると考えています。なぜなら、Shadow DOMがまさにそのための機能を提供してくれており、一方で既存のCSS in JSにはいまだに絶対的な唯一解が無く、進化の余地があると考えられるからです。

CSS in JSに係るエコシステムはWeb Componentsの恩恵を受けて新たな形に進化するでしょう。これが**Web Componentsを従える**ということです。

「そうはいってもWeb ComponentsとかShadow DOMとか使ったことないし、学習コストが……あんな欠点やこんな欠点が……」といった考えをお持ちの方も多いでしょう。それは問題ありません。なぜならこの記事は**次の時代**の話をしているのであって、今すぐWeb Componentsを採用しろと言っているわけではないからです。

実際、Web Componentsはまだ進化の途中です。つい最近話題になった[Declarative Shadow DOM](https://web.dev/declarative-shadow-dom/)には筆者も期待しています。これがあれば、Web ComponentsベースのCSS in JSでSSRをうまいことできるからです（逆に言えば、これがないとWeb ComponentsのSSRが結構厳しいです）。

## 現在のCSS in JS

Web Components時代のCSS in JSの話をする前に、現在のCSS in JSをちょっと見てみましょう。

CSS in JSをやることに悩ましいのは、**スタイルをどうローカル化するか**です。現在のベストプラクティスについては、筆者の意見は次の記事に近いものです（React + styled-componentsが前提）。

- [経年劣化に耐える ReactComponent の書き方](https://qiita.com/Takepepe/items/41e3e7a2f612d7eb094a)
- [ブログに CSS in JS 環境下での スタイル分離リファクタリングを施してみた](https://blog.ojisan.io/s-c-refactor)

1番目の記事から**DOM層**と**Style層**のコンポーネント例を引用します。


```tsx
// DOM層
const Component: React.FC<​Props> = props => (
  <div className={props.className}>
    <button onClick={props.handleClick}>
      {props.flag ? 'click me' : 'CLICK ME'}
    </button>
  </div>
)

// Style層
const StyledComponent = styled(Component)`
> button {...}
`
```

すなわち、DOM構造のみを担当するコンポーネントを用意し、そのDOM構造に対してStyle層でスタイルを付けるというやり方です。この方法の利点としては、DOMの構造に対してスタイルを一括して書くことができる点です。つまり、上の例で言えば、`Component`が定義する`div`と`button`のスタイルは両方とも`StyledComponents`にまとまっています。CSSは、`display: flex`などに代表されるように、親要素と子要素のスタイルが協調しながらデザインを実現することがあります。それに対応する自然な形は、このように単体の要素ごとではなくある程度の大きさのDOM構造に対してまとめてスタイルを書く形でしょう。

ちなみに、筆者はこちらの方が好みです（`className`ベースのデータ受け渡しが煩わしいので）。これはこれで、タグ名が一箇所だけ別の場所に行ってしまうという欠点を持っているのですが。

```tsx
// DOM層
const Component: React.FC<​Props> = props => (
  <Wrapper>
    <button onClick={props.handleClick}>
      {props.flag ? 'click me' : 'CLICK ME'}
    </button>
  </Wrapper>
)

// Style層
const Wrapper = styled.div`
  & > button {...}
`
```

ただ、筆者の考えでは、これらはまだ完成形にたどり着いていません。その理由は主に2つあります。

まず、本来一体のものである「DOM構造」と「それに対するスタイル」を2層に分けて記述しなければいけないこと。そして次に、**スタイルの漏洩**に気をつけないといけないことです。スタイルの漏洩については、先の記事に次のような記述があります。

> `>`による、children への指定漏洩防御も忘れない様にします。

つまり、先の例で`& > button`と書いてあるところをうっかり`& button`のようにしてしまったら、`Wrapper`が関知しない部分（外から子要素として与えられた部分）の`button`にもスタイルが当たってしまうので、それをしないように気をつけなければならないということです。

この2つの問題が、Web Components時代には解決されます。

## Web Components時代のCSS in JS

Shadow DOMを使えば、先ほどのコンポーネントはおおよそ次のような構成になるでしょう。まず、`Component`コンポーネントのShadow DOM内に次のHTMLを入れます。

```html
<style>
button {
  ...
}
</style>
<div>
  <button>
    <slot></slot>
  </button>
</div>
```

これを使う側はこんな感じで使います。元のコンポーネントとロジックの位置がちょっと違っていますが、ロジックはスタイルから完全分離した方がいい（Shadow DOMベースのやり方の場合はそちらの方が向いている）ためこの形になっています。

```tsx
<Component>{
  flag ? 'click me' : 'CLICK ME'
}</Component>
```

こうすると、「HTML構造」と「それに対するスタイル付け」をまとめてShadow DOMの中に放り込むことができ、両者を別々に書かなければならなかった問題が解消します。また、スタイルの漏洩に関しても、Shadow DOMならば鉄壁です。最も簡潔な記述ながら最も堅牢です。それどころか、もっと複雑なスタイルならば、クラスも使い放題です。

```html
<style>
.submitButton {
  ...
}
</style>
<div>
  <button class="submitButton">
    <slot></slot>
  </button>
</div>
```

クラス名が他と被ることなんて心配する必要はありません。なぜなら、Shadow DOMの外に`.submitButton { ... }`というスタイルがあったとしても、Shadow DOMの中には影響を与えられないからです[^note_parts]。なんという理想郷でしょう。

[^note_parts]: 一応、`::parts`を用いてShadow DOMの外のスタイルを中に適用することはできます。ただし、これは許可制です。つまり、Shadow DOMの内側から「この要素に外からスタイルを当てることを許可する」という宣言をする必要があります。

ただ、まだ筆者が考えられていない点もあります。元のコンポーネントにあった`onClick`が上のコンポーネントからは消えていますね。CSS in JSという文脈でイベントをどう扱うのが最も自然かについてはまだいまいち答えを出せていません。考えている途中です。まあ未来の話ですから、その未来が来るまでに考えておけば大丈夫ですね。

### `react-wc`

実は、上記の考え方をもとにすでに`react-wc`というライブラリを作ってしまいました。

- [react-wc: Web ComponentsとReactで実現するCSS in JSの形](https://blog.uhy.ooo/entry/2020-10-03/react-wc/)

一定の使い道ならばすでに利用可能ですが、どちらかといえばまさに“未来”を担う次世代のCSS in JSライブラリを目指しています。詳しいことは当該のブログ記事を見ていただきたいのですが、CSS in JSの用途には次のように使用可能です。Shadow DOMに突っ込みたい物を文字列で指定するだけというとても単純な仕様です。Shadow DOM内はアップデートされることを想定しておらず、そのようなものは`slot()`（`<slot></slot>`に相当）で渡す必要があります。

```tsx
import { html, slot } from "react-wc";
// Reactコンポーネントが作られる
const Component = html`
  <style>
  .submitButton {
    ...
  }
  </style>
  <div>
    <button class="submitButton">
      ${slot()}
    </button>
  </div>
`;

// Reactコンポーネントとして使える
<Component>{flag ? "click me" : "CLICK ME"}</Component>
```

## まとめ

この記事では、Web Componentsを前提としたCSS in JSの未来を予言しました。現状のCSS in JSの諸問題がShadow DOMによって解決されるのですから、準備さえ整えばその方向に進んでいくだろうというのが筆者の予言です。また、Web ComponentsはReactやVueを廃れさせるのではなく、（少なくともしばらくの間は）ViewライブラリがWeb Componentsを利用する形で進化していくだろうと予想しています。

今はまだWeb ComponentsやShadow DOMは目新しいと感じる人が多いかもしれませんが、これらはWeb標準の一部なのですから、言わば「生のDOM」と同じ立ち位置です。今はまだ……とWeb Componentsを避けていけば、その先にあるのは「React/Vueは使えるけど生のDOMは使えない人」という未来です（それがいいことなのか悪いことなのかは人によって意見が分かれるでしょうが、筆者は悪いと思っています）。

そろそろ、Web Componentsを見据えた次の時代について考え始めてもいいのではないでしょうか。筆者は考え始めたので先ほど紹介した[react-wc](https://github.com/uhyo/react-wc)を作りました。今後もWeb Componentsの進化に合わせてreact-wcも進化していくでしょう。今のうちにreact-wcを褒めちぎったりcontributeしておいたりしたら次の時代の先駆者になれるかもしれませんよ。