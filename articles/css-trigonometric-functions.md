---
title: "CSSに三角関数が欲しくなった日"
emoji: "📐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["css"]
published: true
---

[CSS Values and Units Module Level 4](https://drafts.csswg.org/css-values-4/)では、CSSから使える数学関数として三角関数（`sin()`や`cos()`など）が定義されています。まだWorking Draftということもあり、現在のところこれらをサポートしているのはSafariのみのようですが、情報自体は数年前から出ており、目ざとい人やサイトからは紹介されています。

ところが、あまり具体的な用例が紹介されているのは見ません。そこで、この記事では筆者が三角関数を使いたくなった事例を紹介します。

具体例は、下記のような**スクロールアニメーションを持つ斜めのグラデーション**です。これをどう実装するか考えてみましょう。

@[codepen](https://codepen.io/uhyo/pen/RwQryyJ?default-tab=result)

# 斜めのグラデーションの表現方法

止まっている斜めのグラデーションをCSSで表現すること自体は簡単です。次のように`repeating-linear-gradient()`を使って斜めのグラデーションの画像を生成し、それを`background-image`に設定するだけです。このように、`repeating-linear-gradient`は角度を指定することができ、その方向へ繰り返しのグラデーションを生成します。今回の指定は`75deg`で、これは真上から時計回りに75度回った角度を意味します。

```css
.area {
  width: 200px;
  height: 200px;
  background-image: repeating-linear-gradient(75deg, #555555 0, #999999 25px, #f3f3f3 40px, #999999 65px, #555555 100px);
}
```

# スクロールアニメーションの実装

では、これにアニメーションを追加しましょう。上のサンプルでは、右から左に流れているように見えます。そこで、アニメーションで`background-position-x`をスクロールさせれば良さそうです。

```css
.area {
  width: 200px;
  height: 200px;
  background-image: repeating-linear-gradient(75deg, #555555 0, #999999 25px, #f3f3f3 40px, #999999 65px, #555555 100px);
  background-size: 400px 200px; /* ← とりあえずボックスのサイズの2倍を確保 */
  animation: scroll-x 1.5s linear infinite; /* ← アニメーションを追加 */
}

@keyframes scroll-x {
  0% {
    background-position-x: 0;
  }
  100% {
    background-position-x: /* ??? */;
  }
}
```

ここで問題が起こりますね。アニメーションが1周したとき（`100%`）の`background-position-x`をいくつにすればいいでしょうか。ここを間違えてしまうとスムーズなアニメーションになりません。例えば`-100px`は間違いで、次のようにアニメーションの繰り返し時に画像が飛んでしまいます。

@[codepen](https://codepen.io/uhyo/pen/GRQoGKz?default-tab=result)

きれいにアニメーションが繰り返すようにするためには、`background-position-x`が`0`のときの画像と描画が完全に一致するように`/* ??? */`の部分を設定する必要があります。

ここの正解は **-103.5276180410083px** です。

```css
@keyframes scroll-x {
  0% {
    background-position-x: 0;
  }
  100% {
    /* ↓こうするときれいなアニメーションになる！ */
    background-position-x: -103.5276180410083px; 
  }
}
```

もうお分かりと思いますが、この値を計算するために三角関数を使いました。

## 三角関数によるスクロール量の計算

上記の約-103.5 pxという値の計算方法を考えましょう。次の図をご覧ください。

![直角三角形の図。斜辺の長さがx、角の一つが75°、長辺の長さが100 pxである。100 pxの辺はCSSによるグラデーションの周期と一致している。](/images/css-trigonometric-functions/figure1.png)

図に書かれているのは直角三角形です。`repeat-linear-gradient()`の定義をよく見ると周期が`100px`とありますが、これは上の図で100pxと書かれている辺に相当します。一方で、`background-position-x`に書くべき長さは図中にxと書かれている長さです。アニメーションの1週でxだけ水平方向にスクロールさせることで、グラデーションがちょうど1周期分移動したように見せることができます。

図からわかるように、$x = \frac{100\ {\rm px}}{\sin(75\ {\rm deg})}$ となります。これを計算した結果が約103.5 pxです。

ということで、三角関数を使うことで上記のCSSは次のように書けるでしょう。

```css
@keyframes scroll-x {
  0% {
    background-position-x: 0;
  }
  100% {
    background-position-x: calc(-100px / sin(75deg));
  }
}
```

計算後の生の値を書くのに比べて意図が分かりやすくなっています。

また、些細なことではありますが数値の精度を気にかける必要がなくなります。というのも、上記の -103.5276180410083px という数値が小数点以下13桁なのはどういう意味があったのでしょうか。実は、13桁であることはCSS的には特に意味がありません。なぜなら、CSS仕様書には次のように書かれており、CSS上数値の精度に関する規定がないからです。

> CSS theoretically supports infinite precision and infinite ranges
>
> [CSS Values and Units Module Level 4. 5.1. Range Restrictions and Range Definition Notation](https://drafts.csswg.org/css-values-4/#numeric-ranges)

ここで小数点以下13桁（合計16桁）にしたのは、これはおおよそdouble （64ビット浮動小数点数）で表せる精度であり、これくらいあれば十分正確だろうという何となくの判断からです。しかし、何やら中途半端な気持ちになります。

これを`calc(-100px / sin(75deg))`と書くことで精度の問題が無くなり、処理系がサポートする上限の制度を得ることができてすっきりします。え？ 実用上0.1 px単位で十分？　そうですね……。

# まとめ

CSSで三角関数とかいつ使うの？　と言われたらぜひこの記事を紹介してみてください。