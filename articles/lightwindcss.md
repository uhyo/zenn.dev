---
title: "Tailwind CSSからクラス名覚えにくさを消したらどうなる？　こうなった"
emoji: "🌀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["css", "react"]
published: true
---

[Tailwind CSS](https://tailwindcss.com/)はCSSフレームワークの一つで、あらかじめ用意されたクラス名を組み合わせてデザインを構成するというUtility-Firstというコンセプトが特徴です。

筆者は、Tailwind CSSの利点を活かしつつ、「Tailwind CSS特有のクラス名を覚えなければいけない」という問題を解消できないか試してみました。この記事では、その結果としてできたの**Lightwind CSS**を紹介します。

https://github.com/uhyo/lightwindcss

## Tailwind CSSの利点と欠点

[Tailwind CSSの公式サイト](https://tailwindcss.com/)のサンプルを次に引用しますが、Tailwind CSSでは次のようにスタイルを記述します。

```html
<figure class="md:flex bg-gray-100 rounded-xl p-8 md:p-0">
  ...
</figure>
```

これらのクラスは次のような意味です。

- `md:flex` → `@media (min-width: 768px) { display: flex; }`
- `bg-gray-100` → `background-color: #F3F4F6;`
- `rounded-xl` → `border-radius: 0.75rem;`
- `p-8` → `padding: 2rem;`
- `md:p-0` → `@media (min-width: 768px) { padding: 0; }`

`display:flex;`を`flex`で表すことができたり、`padding`が`p`になっていたりと短く書けるのが特徴です。

[Tailwind CSSの公式サイト](https://tailwindcss.com/docs/utility-first)によれば、Utility-FirstなCSSフレームワークには以下のような利点があります。

1. **自分で要素ごとのクラス名を考える手間が省ける。**従来の典型的な方式では要素に対してその要素を表すクラス名（例えば`user-image`とか）を付けてそれに対してスタイルを書く必要があった。
2. **CSSが肥大化しない。**従来の方式だと前述の各クラスに対してそれぞれCSSが書かれるので、要素が増えるたびにCSSの量が増える。Tailwind CSSならば、いくら要素が増えても常に一定のクラス名を使用するので量が増えない。
3. **スタイルがローカルになる。**従来の方式では、一度定義したクラス名を複数の場所で使ったり、あるいは`.cls div`のようなセレクタを用いてクラスが指定されている要素以外に間接的な影響を与えたりする可能性がある。こうなるとCSSの編集の影響範囲が予測できなくなってしまう。Tailwind CSSでは要素に直接クラス名を与え、クラス名は常にその要素にのみ影響を与えるので、編集の影響範囲を制御できる。

一方で、筆者としては、Tailwind CSSの欠点として**CSSなのにCSSではない**ことを挙げたく思います。先ほどの例にあったように、Tailwind CSSではCSSプロパティ名や値を省略してできたクラス名を用います。これはCSSそのものではないので、CSSの知識に比べてTailwind CSSの知識を身に着ける必要があります。とはいっても、例えば`p-8`の`p`が`padding`という意味であり、`padding`がどういう挙動をするのかを理解していないとTailwind CSSを使いこなすことはできません。つまり、Tailwind CSSを使っていても生のCSSの知識は依然として必要なのです。さらに、Tailwind CSSはCSSの機能の全てを網羅しているわけではありません。最新のCSSの機能をフルに使いたい場合は、Tailwind CSSだけでは事足りなくなります。

## Lightwind CSSはCSSをそのまま使う

筆者が開発した**Lightwind CSS**は、Tailwind CSSと同じ利点が得られるようにしつつ、CSSをそのまま使うように修正しました。Lightwind CSSでは例えばこのようにCSSを書きます。

```tsx
<div
  className={css`
    display: flex;
    flex-flow: row nowrap;
    justify-content: center;
  `}
>
  <main
    className={css`
      display: flex;
      flex-flow: column nowrap;
      justify-content: center;
      align-items: center;
    `}
  >
    Hello, world!
  </main>
</div>
```

このように、要素に対して直接CSSを記述します。プログラムとしては`` css`...` ``のなかにCSSを書くと、そのCSSに対応するクラス名が返されます。

この仕組みは[styled-components](https://styled-components.com/)にもちょっと似ていますね。ただし、こちらはスタイルが適用されたコンポーネントを作るのではなく、直接クラス名を得るAPIとなっています。これはTailwind CSSの「**自分で要素ごとのクラス名を考える手間が省ける**」という利点を得るためです。styled-componentsの場合もコンポーネント名を考える必要がありましたが、このように直接`css`を使うことで名前を考える必要が無くなります。また、「**スタイルがローカルになる**」という利点も生きています。Tailwind CSSの場合と同様に、スタイルがDOMと直接結びついていて影響範囲が明確です。

Lightwind CSSは、以上のようなコードをプロダクションビルド時に最適化する機能を持っています。Lightwind CSSにより、全ページのスタイルを集約した1つのCSSファイルが生成されます。上のコードに対応するCSSはこのようになるでしょう。

```css
.a {
  display: flex;
  justify-content: center;
}
.b {
  flex-flow: row nowrap;
}
.c {
  flex-flow: column nowrap;
  align-items: center;
}
```

そして、先ほどのJSXはプロダクションビルドで次のように変換されます（Babelを用います）。

```tsx
<div className="a b">
  <main className="a c">Hello, world!</main>
</div>
```

ここで特に注目すべきは、複数の`` css`...` ``に対して横断的な最適化が行われ、2つの`css`呼び出しで共通する`display`と`justify-content`が`a`というクラスに切り出されて2つの要素の両方で使われているという点です。別々のファイルに存在する`css`も全て一緒くたに最適化されます。この方式により、Tailwind CSSの利点の一つである「**CSSが肥大化しない**」をLightwind CSSでも享受することができます。同じようなスタイルをいくら書いても同じクラス名にまとめられるので、CSSが肥大化しません。

まとめると、Tailwind CSSは「先に最適化されたクラス名がありそれらを使う」というアプローチでしたが、Lightwind CSSでは「先に生のCSSがあってそれらから最適化されたクラスめいを生成する」という逆向きのアプローチとなります。この方法により、Lightwind CSSは先に紹介したTailwind CSSの3つの利点を全て残したまま、Tailwind CSS特有のクラス名を覚えなければいけないという問題を解消しました。

## ただのインラインスタイルとの比較

Lightwind CSSの書き方はインラインスタイルを直接書くのに近いですね。インラインスタイルを直接書くというのは次のようなやり方です（Reactの場合）。Lightwind CSSは、一見するとインラインスタイルをオブジェクトではなく文字列で書けるようにしただけに見えます。

```tsx
<div
  style={{
    display: "flex",
    flexFlow: "row nowrap",
    justifyContent: "center"
  }}
>
  ...
</div>
```

実は、先ほど紹介したTailwind CSSのドキュメントにも「生のCSSを直接書いちゃだめなの？」 (_Why not just use inline styles?_) というセクションがあり、以下のような説明がされています。

1. Tailwind CSSでは`0.75rem`といった具体的な数値を使ってCSSを書く代わりに、`p-8`や`rounded-xl`といった抽象的な名前のクラスを用いる。これにより、あらかじめ定義されたデザインシステムの恩恵を受け、一貫性のあるデザインを実現できる。
2. インラインCSSではmedia queryを使用できないが、Tailwind CSSでは`md:flex`のようなmedia query付きのクラス名が提供されており、クラス名だけでレスポンシブなデザインが可能である。
3. インラインCSSでは`:hover`や`:focus`といった擬似クラスが指定できないが、Tailwind CSSでは`hover:bg-purple-700`のような各種擬似クラス用のクラス名が提供されており、クラス名だけで擬似クラスも使用できる。

Lightwind CSSはこれらの問題のうち、2と3はネストしたルールのサポートにより解決しています。styled-componentsなどでお馴染みの`&`を使うもので、次のように書くことができます。

```tsx
  <main
    className={css`
      display: flex;
      flex-flow: column nowrap;
      justify-content: center;
      align-items: center;

      &:hover {
        opacity: 0.5;
      }
    `}
  >
    Hello, world!
  </main>
```

1については、「生のCSSを書きたい」という思想から、デザインシステムの提供はLightwind CSSの責務に含めていません。一貫したデザインを用意するのは利用者の仕事です。Tailwind CSSが提供するデザインシステムに満足できない場合はこれはむしろ利点にもなるでしょう。

一貫性が必要ならば、CSS Variablesなどを用いたthemingも有用でしょう。Lightwind CSSがCSS Variablesをうまく活用するための機能を提供するという方向性はありかもしれません。

### 注意点

Lightwind CSSは上述のようなネストしたCSSをサポートしていますが、これに関して一点気をつけなければならないことがあります。次のようなCSSも書くことができ、こうするとTailwind CSS / Lightwind CSSの利点の一つであり「CSSのローカル性」が崩れてしまいます。

```tsx
  <main
    className={css`
      display: flex;
      flex-flow: column nowrap;
      justify-content: center;
      align-items: center;

      /* この中のp要素全てに適用される！ */
      p {
        color: red;
      }
    `}
  >
    Hello, world!
  </main>
```

見方によっては、Lightwind CSSはTailwind CSSよりも自由度が高いということもできるでしょう。Tailwind CSSは自由度を敢えて抑制しているところに魅力があるとも言えます。

Lightwind CSSでも自由度の抑制が必要ならば、stylelintを使うなど追加の施策が必要です。むしろ、Lightwind CSSは敢えて制限を減らすことで自身の責務を単純化したと言うこともできます。

## まとめ

この記事では、Tailwind CSSと同じ利点を保ったまま、独自のクラス名ではなく生のCSSを書けるようにしたCSSフレームワークである**LightwindCSS**を紹介しました。Lightwind CSSの特徴は、生のCSSが書かれた全てのファイルをグローバルに解析して最適なクラス定義を生成できる点にあります。

Tailwind CSSはいいけど独自のクラス名を覚えたくないという方や、Tailwind CSSはお節介すぎるのでもっとカスタマイズしやすいツールが欲しいという方には合うかもしれません。

いいねと思ったらぜひLightwind CSSにスターをよろしくお願いします。

https://github.com/uhyo/lightwindcss