---
title: "『プロを目指す人のためのTypeScript入門』読者が最新情報にキャッチアップできる記事"
emoji: "🫐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

こんにちは。先日発売された『**プロを目指す人のためのTypeScript入門**』は、発売日の最新バージョンである**TypeScript 4.6**に対応しています。

https://gihyo.jp/book/2022/978-4-297-12747-3

そこで、この記事では読者に向けたアフターサポートとして、本の発売時から現在までに増えた機能や変わったところをご紹介します。

現在のTypeScript最新版は**4.7**です。

:::message
この記事では、本に載せるレベルの機能追加や、本に書いてあることが変わってしまうところを抜粋して取り扱います。TypeScriptの更新は大小色々ありますが、小さいものはこの記事では省いています。
:::

# TypeScript 4.7での更新

公式アナウンス: https://devblogs.microsoft.com/typescript/announcing-typescript-4-7/

## Node.js向けES Modulesサポートの追加

TypeScript 4.7最大の話題はこちらです。本書の第1章では、tsconfig.jsonの設定項目について次のように説明しました。

```json
  "module": "esnext"
```

TypeScript 4.7では、Node.js環境での開発により適した設定項目として以下の2つが追加されました。

```json
  "module": "node16"
```

```json
  "module": "nodenext"
```

現在のところ`node16`と`nodenext`には違いがありません。将来のNode.jsバージョンでNode.jsのES Moduleサポートが変化した場合、`node16`は現状のままで`nodenext`は新しいバージョンのNode.jsの仕様に追随することになります。

この新しい設定を適用した場合、本書で解説した内容は次のように変わります。

### ファイルがスクリプトかモジュールかの判定の変化

`module`が`node16`/`nodenext`の場合は、ある`.ts`ファイルがスクリプトかモジュールかの判定方法が従来から変わります。第7章の**コラム34**で解説されているように、TypeScriptファイルはスクリプトとモジュールの2種類があります。従来の判定方法は、「ファイルに`import`/`export`が含まれていたらモジュール」というものです。他の多くの環境では外的要因によってスクリプトかモジュールか決まるのに、TypeScriptでは内的要因によって決まるのが特殊でした。

`node16`/`nodenext`下では、`.ts`ファイルがスクリプトかモジュールかは**package.jsonのtypeフィールド**を参照して決められます[^note_moduleDetection]。

[^note_moduleDetection]: ただし、同時に追加された`moduleDetection`オプションを用いるとこの挙動を変更することができます。

本書の第1章では環境構築の一部として`"type": "module"`として設定していました。この設定下では、Node.jsは`.js`ファイルをモジュールとして扱います。つまり、`.ts`をコンパイルしてできた`.js`は常にモジュールになるのです。

従来のTypeScriptではモジュールの条件を満たさない`.ts`ファイルはスクリプトとして扱われましたから、あるファイルがスクリプトかモジュールかの判断が、TypeScript側と実行環境のNode.js側で齟齬が出る場合がありました。

`node16`/`nodenext`下では、両者がどちらもpackage.jsonのtypeフィールドを参照するようになり、判断が一致します。

### TypeScript用の新しい拡張子

前述のように、Node.jsではpackage.jsonのtypeフィールドを参照して`.js`ファイルがスクリプトかモジュールか決められました。つまり、あるpackage.jsonの支配下においては`.js`ファイルの種類が統一されることになります。たまに、その判断を上書きして、特定のファイルだけスクリプト/モジュールの区分を変更したいことがあります。Node.jsではそのための拡張子として`.cjs`と`.mjs`を用意しています。package.jsonのtypeフィールドがどうであろうと、`.cjs`は常にスクリプトとして、`.mjs`は常にモジュールとして扱われます。ちなみに、cというのはCommonJSを表すとされています。

そして、これらの拡張子に対応するものとして、TypeScriptにも`.cts`と`.mts`という新しい拡張子のサポートが追加されます。これらの拡張子とコンパイル後の拡張子の関係は次の表のようになります。

| TypeScript |   | JavaScript |
| ---------- | - | ---------- |
| `.ts`      | → | `.js`      |
| `.cts`     | → | `.cjs`     |
| `.mts`     | → | `.mjs`     |

TypeScriptにおいても、`.cts`は常にスクリプトとして、`.mts`は常にモジュールとして扱われます。また、本書では`import`宣言においてコンパイル後の拡張子を書かなければいけないと説明しましたが、それは`.cts`や`.mts`でも同じことです。つまり、`.mts`ファイルをインポートしたい場合、次のように拡張子は`.mjs`とする必要があります。

```ts:foo.mts
export foo = "Hello!";
```

```ts:index.mts
import { foo } from "./foo.mjs"; // ← .mjs を使用
```

筆者の経験した例としては、諸事情によりまだpackage.jsonに`"type": "module"`と書けないプロジェクト（Node.jsではこの場合全て`.js`のファイルがスクリプトとして扱われます）において部分的にモジュールを使いたい場合に`.mjs`（およびそのTypeScript版の`.mts`）を重宝します。

### その他の機能

`module`が`node16`/`nodenext`のモードの特徴は、TypeScriptが意思決定にpackage.jsonの内容を使用するということです。これは従来無かったことです。それに伴って、このモードではpackage.jsonに書かれた他の情報もいくつか利用されます。特筆すべきはpackage.jsonの[exportsフィールド](https://nodejs.org/dist/latest-v16.x/docs/api/packages.html#exports)のサポートも入ったことです。

これにより、`foo/bar`（`foo`というパッケージの中の`bar`というエントリーポイント）のようなインポートの取り扱いをより柔軟にすることができます。実務では、どちらかというとこれのサポートを目当てに`module`を`node16`などに変えることが多いでしょう。

さらに、package.jsonのexportsフィールドはNode.js発の機能ですが、webpackなどのバンドラからもサポートがあり、フロントエンド開発などでも利用されています。そのため、フロントエンド向けのTypeScriptプロジェクトでも`module`を`node16`/`nodenext`にする機会が今後あるはずです。フロントエンドのプロジェクトなのに設定に`node`と書くのは違和感があるかもしれませんが、Node.jsに由来するデファクトスタンダードを利用するというように捉えればよいでしょう。

# 具体化式 (Instantiation Expression)

**具体化式** (Instantiation Expression) はTypeScript 4.7で新たに追加された構文です。これは、第4章で説明したような型引数を持つ関数（ジェネリック関数）に対して使用できる式です。

具体化式を一言で言うと、「**関数に型引数だけ与えて呼び出さない**」という構文だと理解できます。

```ts
// ジェネリック関数の定義
function repeat<T>(elm: T, n: number): T[] {
  const result: T[] = [];
  for (let i = 0; i < n; i++) {
    result.push(elm);
  }
  return result;
}

// 関数呼び出し
const arr1 = repeat<number>(1, 3); // [1, 1, 1]
// 具体化式
const repeatNumber = repeat<number>;
```

上の例の最後の行が具体化式の使用例です。`repeat<number>`が具体化式です。これは、`repeat`関数の型引数を`number`と決めるという意味です。変数`repeatNumber`の型を調べてみると、次のようになります。

```ts
(elm: number, n: number) => number[]
```

つまり、`repeat`を呼び出したわけではないので`repeatNumber`は関数のままですが、型引数が消えて`T`が`number`に固定されています。これが、関数に型引数だけ与えるということです。

ちなみに、トランスパイル後のJavaScriptはこうなります。

```js
const repeatNumber = (repeat);
```

型引数が消えるのはあくまで型の話なので、ランタイムでは何も起こらないということを反映しています。

具体化式の使いどころを見つけるのはやや難しめですが、本来汎用的な関数を、特定の用途に向けたものとして再定義するときに有用かもしれません。公式アナウンスでは次のような例が紹介されていました。

```ts
// キーの型と中身の型が固定されたMapのコンストラクタを作成
const ErrorMap = Map<string, Error>;
```

# 型引数に対する変性アノテーション

TypeScript 4.7では型引数に対して変性アノテーション (variance annotation) を書ける機能が追加されました。

## 変性とは

変性 (variance) とは、**共変**や**反変**といった性質のことです。本書では、**4.3.2**「引数の型による部分型関係」などで解説しています。例えば、関数型においては引数の型は反変の位置にある一方、返り値の型は共変の位置にあります。

```ts
// Tは反変の位置にあり、Uは共変の位置にある
type Func<T, U> = (arg: T) => U;
```

上の例では`Func`型が型引数`T`と`U`を持っていますが、実は型引数は変性を持ちます。コメントにある通り、`T`は反変で`U`は共変です。実はTypeScriptコンパイラは内部でこのような情報を計算し、型検査に活用しています。たとえば次の例をご覧ください。最後の行の代入は型安全ではないためエラーになります。

```ts
const func: Func<number, string | number> = (arg: number) => Math.random() < 0.5 ? arg : arg.toString();

// TypeScriptコンパイラさん「Funcの型引数Uは共変だからこの代入はダメだな……
const func2: Func<number, string> = func;
```

このときのエラーメッセージがポイントです。

```
Type 'Func<number, string | number>' is not assignable to type 'Func<number, string>'.
  Type 'string | number' is not assignable to type 'string'.
    Type 'number' is not assignable to type 'string'.
```

このエラーメッセージは`Func`の2番目の型引数を比較して、`string | number`は`string`に代入可能ではないからだめだよと言っています。このとき、エラーメッセージは`Func`の内部構造（関数型）に立ち入っていません。TypeScriptはあらかじめ`Func`の型引数`U`は共変であるという情報を計算しているので、その情報さえあれば`Func`の中身を見に行かなくても型検査ができるのです。

このように、型関数（型引数を持つ型）は型検査における区切りとしても使用されます。

これまでは、型引数の変性はその中身を見てTypeScriptが推論していました。これは、変数の型注釈や関数の返り値の型を書かなかった場合に推論してもらえるのと同じことだと理解できます。

TypeScript 4.7では、型引数の変性を明記できるようになりました。そのための`in`と`out`というキーワードが追加されています。これらのキーワードは次のように型引数に前置する形で使います。`in`は反変、`out`は共変を表します。

```ts
type Func<in T, out U> = (arg: T) => U;
```

`in`と`out`というキーワードは、一般に値に対して入力されるもの（例えば関数の引数）は反変であり、出力されるもの（例えば関数の返り値）は共変であることを反映しています。

これらのキーワードを明記することは、変数の型や関数の返り値の型を明記するのに似ています。つまり、書いたことと実際の実装が食い違っている場合、コンパイルエラーが発生します。例えば、`in`と`out`を逆に書いてしまうと次のようなコンパイルエラーが発生します（個人的にちょっとエラーメッセージの意味が分かりにくい気がしますが）。

```ts
type Func<out T, in U> = (arg: T) => U;
// エラー:
// Type 'Func<sub-T, U>' is not assignable to type 'Func<super-T, U>' as implied by variance annotation.
//   Types of parameters 'arg' and 'arg' are incompatible.
//     Type 'super-T' is not assignable to type 'sub-T'
// エラー:
// Type 'Func<T, super-U>' is not assignable to type 'Func<T, sub-U>' as implied by variance annotation.
//   Type 'super-U' is not assignable to type 'sub-U'.
```

変性アノテーションは書かなくても推論してもらえるものですが、書くことでよりコードが読みやすくなったり、ミスを検知しやすくなったりという効果を得ることができます。
