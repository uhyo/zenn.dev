---
title: "CSSにそのうち導入されそうな@scopeとその関連概念"
emoji: "🔭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["css"]
published: false
---

気がつけばCSSの[@layer](https://developer.mozilla.org/en-US/docs/Web/CSS/@layer)が全てのモダンブラウザに実装完了している今日この頃、みなさまはいかがお過ごしでしょうか。

CSSでは、`@layer`に次ぐ新機能として **`@scope`** が検討されています。最近これについて勉強したのですが、これを取り扱う日本語記事が見当たらなかったので今回ご紹介します。

この記事では、CSS Cascading and Inheritance Level 6のFirst Public Working Draftの内容を紹介します[^note_selector_4]。これは去年12月のバージョンで、より新しいEditor's Draftとして今年4月のものがありますが、特に大きな変更はありませんでしたので、この記事の内容が執筆時点の最新情報だと思って差し支えありません。

[^note_selector_4]: 一部の概念は[Selectors Level 4](https://drafts.csswg.org/selectors-4/)にも依存しています。

https://www.w3.org/TR/2021/WD-css-cascade-6-20211221/

:::message
当然ながら、現在（2022年5月）この記事の内容を実装したブラウザはありません。また、この記事の内容は今後実装されるまでに大きく変化する可能性があります。
:::

## @scopeとは: 基本的な構文

`@scope`とは、次の例のような構文を持つ記法です。

```css
@scope (main) {
  div {
    border: 1px dashed black;
  }
  p {
    margin-block: 1em;
  }
  strong {
    color: red;
  }
}
```

このように、スタイルルールを`@scope (セレクタ) { ... }`という構文で囲むのが基本的に`@scope`の構文です。

`@scope`の`(セレクタ)`部分にマッチした要素のことを**scoping root**と言います。そして、`@scope`のブロック内に書かれたセレクタは、scoping rootの子孫要素に対してのみマッチするようになります。

![文書の木構造を描いた図。mainの配下に三角形状のスコープが広がり、mainの子孫であるdiv, p, strongにはマッチするが、mainの子孫でないpやstrongにはマッチしないことを示す。](/images/css-cascading-6-scope/scope-behavior-1.png)

上の例では、`main`でスコープされた定義の中に`div`, `p`, `strong`がセレクタとして用いられています。よって、これらのセレクタは`div`や`p`や`strong`なら何でもマッチするわけではなく、`main`の子孫である`div`, `p`, `strong`にのみマッチします。

このように、中に書かれたスタイルルールの適用範囲を制限するのが`@scope`の機能です。

上の例の場合、次のように書き換えることもできるでしょう。

```css
/* @scope を使う例 */
@scope (main) {
  div {
    border: 1px dashed black;
  }
  p {
    margin-block: 1em;
  }
  strong {
    color: red;
  }
}

/* @scope を使わない例 */
main div {
  border: 1px dashed black;
}
main p {
  margin-block: 1em;
}
main strong {
  color: red;
}
```

両者は**だいたい**同じです。`@scope`を使う場合と使わない場合には多少違いがありますが、それはこの記事で追々解説します。

ちなみに、scoping rootを指定するセレクタは何でも書くことができます。`main`のような単純なものばかりでなく、次のような指定もできます。

```css
@scope (table.price > tbody) {
  /* ... */
}
```

もしscoping rootに当てはまる要素が複数ある場合は、`@scope`内のセレクタはscoping rootのいずれかの子孫であればマッチします。

## スコープの下限を設定する

`@scope`の特徴的な機能は、**スコープの下限を設定できる**ことです。上の例は下限設定機能を使っていませんでした。そのため、`@scope`を使わなくても似たようなことができます。下限設定機能を使うことで`@scope`の本領が発揮されます。スコープの下限を設定するには、`to`構文を用います。

```css
/* @scope を使う例 */
@scope (main) to (aside.ad) {
  div {
    border: 1px dashed black;
  }
  p {
    margin-block: 1em;
  }
  strong {
    color: red;
  }
}
```

この例を読み下せば、「`main`から`aside.ad`まで」となります。「まで」というのがどういう意味かというのは、やはり木構造の図で見ると分かりやすいでしょう。

![木構造を用いた図。mainの子孫であるdiv, p, strongにはマッチするが、mainの子孫であってもaside.adの下にあるpやstrongはマッチしないことを示している。](/images/css-cascading-6-scope/scope-behavior-2.png)

意味としては、上記の`@scope`が表すスコープの範囲は「`main`の子孫かつ、その中の`aside.ad`の子孫は除く」となります。これは、木構造で見ると、ルートから末端までの道の間で`main`から`aside.ad`までの間にある要素に対してのみマッチすることを示しているということです（`aside.ad`が無いパスの場合は、末端までスコープに含まれます）。

このように、`to`で示された要素はスコープの終端（scoping limit）となり、それより下はスコープの範囲外となります。

## スコープの便利な使用例