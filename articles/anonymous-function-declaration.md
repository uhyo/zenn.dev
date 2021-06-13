---
title: "JavaScriptに密かに存在する“無名関数宣言”"
emoji: "🤳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "ecmascript"]
published: true
---

この記事では JavaScript エンジニアがしてしまいがちなある誤解を紹介し、それがなぜ誤解なのかを解説します。
その誤解とは、「関数宣言には必ず名前が必要である」ということです。これは`export default`の場合に例外が存在しているため、誤解となります。

## JavaScript の関数宣言

JavaScript で関数を作る方法は色々ありますが、その中でも`function`キーワードを用いる方法は初期から存在しています。`function`キーワードを用いて関数を作る場合は**関数式**と**関数宣言**の 2 つに大別されます。関数式はその名の通り式である一方で、関数宣言は文のように使用され、巻き上げ (hoisting) の挙動を持つことが特徴的です。

```js
// 関数式
const func = function (num) {
  return num * 2;
};

console.log(func(100));
```

```js
// 関数宣言
console.log(func(100));

function func(num) {
  return num * 2;
}
```

`function`の後に書く関数名は、関数式の場合は省略して無名関数とできる一方で、関数宣言の場合は関数名を省略すると構文エラーとなります。実際、Google Chrome で試すと「Uncaught SyntaxError: Function statements require a function name」という構文エラーが報告されます。

```js
// 構文エラー
function (num) {
  return num * 2;
}
```

このように、関数宣言として`function`の構文を使う際は関数名が必要であるというのが一般的な JavaScript エンジニアが持つ経験則です。確かに、名前がない関数宣言というのは作った関数を参照する手段が無くなってしまうことからもこれは妥当な制約です。しかし、実はこの「関数宣言には名前が必要である」という制約には一つ例外があるのです。

## `export default`と関数宣言

次に`export default`宣言の話に移ります。これはモジュールから何かを default エクスポートするための構文であり、典型的な構文は`export default 式;`です。例 2 のように何かのインスタンスをエクスポートしてシングルトン的に使うような例を見たことがある方も多いでしょう。

```js
// 例1
export default 123;

// 例2
export default new SomeBigClass();
```

一方で、React のプロジェクトなどでは、次のように`export default`で関数をエクスポートする例も見られます。

```js
export default function (props) {
  // ...
}
```

この構文が今回の本題です。`export default 式;`の類例から見ればこの`function (props) { ... }`は一見関数式のように見えますが、**実はこれは関数式ではなく関数宣言です**。そして、このように`export default`に続く関数宣言では特別に名前なしの関数宣言——すなわち無名関数宣言——が認められているのです。

### 仕様書で確かめる

以上のことを仕様書で確かめてみましょう。https://tc39.es/ecma262/ を参照します。

以下に引用するように、構文定義上`export default`構文には 3 種類あります。

> `export default` HoistableDeclaration[~Yield, ~Await, +Default]\
> `export default` ClassDeclaration[~Yield, ~Await, +Default]\
> `export default` [lookahead ∉ { `function`, `async` [no LineTerminator here] `function`, `class` }] AssignmentExpression[+In, ~Yield, ~Await] `;`

![仕様書のスクリーンショット](https://storage.googleapis.com/zenn-user-upload/ce28c1ffc05c5ad18acb172c.png)

すなわち、`export default`の後ろに HoistableDeclaration が来るもの、`export default`の後ろに ClassDeclaration が来るもの、そして`export default`の後ろに AssignmentExpression `;` が来るものです。HoistableDeclaration は`function`、`function*`、`async function`、`async function*`という 4 種類の`function`宣言を総称する非終端記号です。

`export default function`の形の宣言は、1 番目の HoistableDeclaration に当てはまります。3 番目（後ろに式が来る形）に当てはまる可能性は、lookahead ∉ { ... }という制限によって除去されています（`export default`の直後に`function`や`async function`・`class`といったトークン（列）が来るものが除かれています）。

以上のことが、`export default function`における`function`が関数式ではなく関数宣言であることの根拠となります。

次に、関数宣言（FunctionDeclaration 非終端記号）の定義も見てみましょう。

> FunctionDeclaration[Yield, Await, Default] :
>
> `function` BindingIdentifier[?Yield, ?Await] `(` FormalParameters[~Yield, ~Await] `)` `{` FunctionBody[~Yield, ~Await] `}`\
> [+Default] `function` ( FormalParameters[~Yield, ~Await] `)` `{` FunctionBody[~Yield, ~Await] `}`

![仕様書のスクリーンショット](https://storage.googleapis.com/zenn-user-upload/a85e811b562bf64f711cdaf1.png)

このように、FunctionDeclaration の構文には 2 種類あります。前者は`function`トークンの後ろに BindingIdentifier がありますが、後者にはありません。これらがそれぞれ、関数名がある関数宣言と関数名が無い関数宣言に対応します。ただし、後者には最初に\[+Default\]と書いてあるのが見て取れます。実は FunctionDeclaration 非終端記号は Yield, Await, Default という 3 つのパラメータでパラメトライズされており、そのうち Default というフラグが立っている場合にのみ 2 つ目の定義が有効（使用可能）という定義になっています。

`export default`の構文定義に立ち返ってよく見てみると、 `export default` HoistableDeclaration[~Yield, ~Await, +Default] とあり、HoistableDeclaration に +Default としてパラメータを渡しています。これが Default パラメータをオンにするという意味です（+は有効、~は無効を表します。他に?というのがあり、これは自身の同名パラメータを引き継ぐという意味です）。HoistableDeclaration に渡された Default パラメータはそのまま FunctionDeclaration に引き継がれます。

このことにより、`export default`の後ろの関数宣言においては無名バージョンの関数宣言が使用可能となるのです。他に Default フラグが立つところはありませんから（これは仕様書を全文検索してみてば分かります）、無名関数宣言が可能なのは`export default`の場合のみです。

## 関数式と関数宣言の違いが現れる例

以上で解説した通り、`export default function`の形における`function`は関数式ではなく関数宣言です。このことから帰結する少し面白い例を紹介したのが以下のツイートです。

https://twitter.com/uhyo_/status/1403661230175256589

ツイートの画像にあるコードを示します。

```js
const expr = function() {} 123;
//                         ^^^
//          ここで構文エラーが発生

export default function () {} 123;
```

`const`による変数宣言と`export default`の場合で、どちらも`function() {} 123;`という同じコード列であるにもかかわらず、`const`の場合は`123`で構文エラーが発生する一方`export default`の場合は構文エラーが発生しません。

この違いは、`const`の方では`function`が関数式であるのに対して`export default`の方では`function`が関数宣言であることから発生します。

変数宣言は`const 変数 = 式;`という構文ですから[^note_1]、`function() {} 123`の部分が式とならなければなりません。関数式の場合は`function() {}`でひとつの式となりその後ろに`123`が続くような構文は存在しないため、`123`は構文エラーとなります。

[^note_1]: 分割代入などもありますが今回は関係ないので省略しています。

一方で、`export default`の場合も`function () {}`までで関数宣言が終わるのは同じですが、上記の`export`宣言の定義をよく見ると分かるように、`export default`のあとが関数宣言である場合はセミコロンが必要ありません。よって、`}`の時点で`export`宣言が終了したものと見なされます。そのため、`123;`は`export`宣言とは別の単独の文と見なされ、（意味はないものの）構文的に正しいプログラムとなります。

まとめると、式か宣言かによってセミコロンの要不要が異なり、それが`123`が構文エラーになるかどうかを左右しているのです。

## まとめ

「JavaScript の関数宣言には**必ず**関数名が必要」と言ってしまうと誤りなので、気をつけましょう。

余談ですが、`class`宣言の場合も同様に、`export default`の場合のみ無名クラス宣言が可能です。

ちなみに、`export default`の無名関数宣言で作られた関数は、自動的に`default`という関数名が与えられます。詳しくは[JavaScript の関数名の全て](https://qiita.com/uhyo/items/2a2d67abd56419ea21b8)をご覧ください。
