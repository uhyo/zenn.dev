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

現在のTypeScript最新版は**4.9**です。

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

## 具体化式 (Instantiation Expression)

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

## 型引数に対する変性アノテーション

TypeScript 4.7では型引数に対して変性アノテーション (variance annotation) を書ける機能が追加されました。

### 変性とは

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

# TypeScript 4.8での更新

公式アナウンス: https://devblogs.microsoft.com/typescript/announcing-typescript-4-8/

TypeScript 4.8では、特筆すべき言語機能の追加はありませんでした。一方で、型推論周りの改善がひとつあり、`unknown`型の使い勝手が向上しました。

具体的に言えば、`unknown`に対して`!== null`や`!== undefined`といった絞り込みが効くようになります。

```ts
function useUnknown(value: unknown) {
  if (value !== null && value !== undefined) {
    // TypeScript 4.7: valueはunknownのまま
    // TypeScript 4.8: valueは{}型
  }
}
```

このように、`unknown`型からnullとundefinedの可能性を除外した場合、TypeScript 4.8からは`{}`型に変化します。`{}`型は空のオブジェクト型です。これは本にも説明があったように、nullとundefined以外の任意の値に当てはまる型です。とりあえず条件分岐でnullとundefinedを除外するのは頻出の処理ですから、その場合に対応が入るのは嬉しいですね。

より詳しい説明を以下の記事でしているので、あわせてご参照ください。

https://zenn.dev/uhyo/articles/typescript-4-8-type-narrowing

# TypeScript 4.9での更新

公式アナウンス: https://devblogs.microsoft.com/typescript/announcing-typescript-4-9/

## `satisfies` 演算子の追加

TypeScript 4.9において特に注目すべき機能追加は`satisfies`演算子です。

この演算子の構文は`式 satisfies 型`です。構文の見た目としては`as`に似ていますね。`satisfies`は`as`とは異なり安全な機能で、機会を見つけてぜひ活用したいものです。

この構文は、ランタイムの挙動はとくに持ちません。つまり、ランタイムの挙動はただの`式`と同じです。`satisfies`の役目は追加の型チェックを提供することにあります。この点で、`const`宣言や関数の返り値に対する型注釈と似た立ち位置にあります。

基本的な`satisfies`の挙動は単純で、`式`が`型`に代入可能であることをチェックするという意味になります。

```ts:コンパイル可能な例
// 3はnumberに代入可能なのでOK
const foo = 3 satisfies number;
// { pika: "chu" } は Record<string, string>に代入可能なのでOK
const obj = { pika: "chu" } satisfies Record<string, string>;
```

```ts:コンパイルエラーが発生する例
// エラー: Type 'number' does not satisfy the expected type 'string'.
const foo = 3 satisfies string;
// エラー: Type 'string' is not assignable to type 'number'.
const obj = { pika: "chu" } satisfies Record<string, number>;
```

### 変数の型注釈との違い

上の例を見ると、`satisfies`を使わなくても次のようにすれば良いようにも思えます。

```ts
// これでも同じ？
const foo: number = 3;
const obj: Record<string, string> = { pika: "chu" };
```

実際、書いた式が目的の型に合っているかをチェックしたければ上のように書けます。

実は、`satisfies`は上記のように変数の型注釈を使う場合に比べて利点があります。それは、**式から推論された具体的な型が保存される**という点です。このことを確かめてみましょう。

```ts
// foo1 は number 型
const foo1: number = 3;
// foo2 は 3 型
const foo2 = 3 satisfies number;
```

変数の型注釈を使う場合と`satisfies`を使う場合とを比べると、変数`foo1`は型が`number`となっていて（型注釈でそう書いてあるから当然ですが）、その中身が具体的には3であるという情報が消えています。一方、`foo2`では変数の型が`3`（数値のリテラル型）となっていて、`3`という式から推論された型が生きています。

このように、`satisfies`を使うことで式の型推論の結果をフルに活用することができます。上の例だといまいちありがたみが分かりませんが、次の例ではより分かりやすいでしょう。

```ts
const obj1: Record<string, string> = { pika: "chu" };
const obj2 = { pika: "chu" } satisfies Record<string, string>;

obj1.pika; // string | undefined （noUncheckedIndexedAccess: trueの場合）
obj2.pika; // string

obj1.pikachu; // string | undefined
obj2.pikachu; // コンパイルエラー
```

このように、`obj1`では「`pika`プロパティが存在する」という情報が型から消えていますが、`obj2`では残っています。

### `satisfies`の有用な使い方

実は、`satisfies`は本のコラムで説明した「値を最上位の事実とする」パターンと組み合わせるのが特に有用です。上の例では`typeof obj2`は`{ pika: string; }`となり、`obj1`に比べるとキー名の情報が残っている分だけ型情報がより有用です。この型を起点に他のところで例えば`keyof typeof obj2`などいった形で発展させられます。

このように式の型推論の結果を`typeof`で取り出したい場合は、当然ながらその変数には型注釈を付けられませんでした。そのため、従来は根本となる定義（上の例の`obj2`）に対して型チェックを行いにくいのが問題でした。

例えば、`obj2`は「何らかのキーに対して文字列が入っている」という形のオブジェクトになっていることが期待されていたとします。つまり、`{ pika: 12345 }`のような定義だとしたらコンパイルエラーで弾きたかったとしましょう。これはまさに`satisfies`がやってくれることです。

```ts
const obj2 = { pika: "chu" } satisfies Record<string, string>;
```

この例では`obj2`は「`Record<string, string>`型を満たす」という、すなわちすべてのキーの型は`string`型であるという制約を課されれています。従来、`obj2`の型情報を損なわずに同じことをするのは少々面倒でした。以下のようなやり方がありました。

```ts
const obj2 = { pika: "chu" } satisfies Record<string, string>;
// 別の変数に入れてチェック
const _check: Record<string, string> = obj2; 
```

```ts
// 制約チェック用の関数をかませる
const obj2 = constraintCheck({ pika: "chu" });

function constraintCheck<T extends Record<string, string>>(value: T): T {
  return value;
}
```

これらのやり方は回りくどくて、読み手に意図が伝わりにくいものでした。TypeScript 4.9での`satisfies`の導入により、このようなユースケースで分かりやすいコードを書くことができるようになったのです。

### `satisfies`とContextual Typing

本を読んだ方は、Contextual Typingが発生する条件についてご存じでしょう。例えば、`const 変数: 型 = 式`における`式`に対してContextual Typingが発生します。

実は、`式 satisfies 型`においても`式`に対してContextual Typingが発生します。このことは、次のような例で確かめられます。

```ts
type PokemonTrainer = {
  choose: (name: string) => string;
};

const satoshi = {
  choose: (name) => `${name}！　きみにきめた！`
} satisfies PokemonTrainer;
```

上の例では、Contextual Typingの効果として、関数式の引数の型注釈を省略できるという効果が表れています。

他の例としては、Contextual Typeとしてリテラル型が与えられた場合には型推論の結果もリテラル型になるという効果もあります。

```ts
type PikaObj = { pika: "pika" | "chu" };

// obj1 は { pika: string } 型
const obj1 = { pika: "chu" }
// obj2 は { pika: "chu" } 型
const obj2 = { pika: "chu" } satisfies PikaObj;
```

このようにすると、`obj1`と`obj2`は`satisfies`があるかどうかしか違いがないところ、`pika`プロパティの型が異なっています。これもContextual Typingの効果です。

## `accessor` キーワードの追加

TypeScript 4.9ではもう一点構文の追加があります。それが`accessor`です。これはクラス宣言の中で使える構文であり、フィールド定義の前に付けることができます。

```ts
class A {
  // accessor なし
  foo: number = 1;
  // accessor あり
  accessor bar: number = 2;
}
```

`accessor`の効果は、`accessor`付きで宣言されたプロパティの実態をアクセサプロパティ（`get`/`set`で宣言されたプロパティ）にするというものです。つまり、上の`bar`を`accessor`構文を使わない形で書き直すと次のようになります。

```ts
class A {
  #bar_accessor_storage: number = 2;
  get bar(): number {
    return this.#bar_accessor_storage;
  }
  set bar(value: number) {
    this.#bar_accessor_storage = value;
  }
}
```

このことから分かるように、フィールドに`accessor`キーワードを付けても基本的な挙動の違いは特になく、同じように動作します。ただし、細かな違いはあります。本書の範囲を超えますが、例えばアクセサプロパティは定義の実態がインスタンス上ではなくprototype上にあります。

```ts
class A {
  foo: number = 1;
  accessor bar: number = 2;
}

const a = new A();

console.log(Object.hasOwn(a, "foo")); // true
console.log(Object.hasOwn(a, "bar")); // false
```

プロパティ定義の実態が`get`/`set`なのか、あるいはプロパティ定義がインスタンス上にあるかprototype上にあるかというのは、普段意識する必要がない細かい違いです。そのため、今のところ皆さんが`accessor`をわざわざ使う場面はなさそうです。この構文が意味を成すためには、TypeScriptにデコレータ[^note_legacy_decorator]が実装されるまで待つ必要があります。

[^note_legacy_decorator]: TypeScriptにすでに実装されているレガシーなデコレータ機能ではなく、ECMAScriptでStage 3まで進んだほうのデコレータです。


## `in`演算子の機能強化

TypeScript 4.9では、in演算子に新しい型絞り込み機能が追加されました。それは、型上で存在しないプロパティに対して`in`で絞り込みを行うことで、今後その名前でのプロパティアクセスが可能になるというものです。次がその例です。

```ts
function checkName(obj: unknown): obj is { name: string } {
  if (obj === null || typeof obj !== "object") {
    return false;
  }
  if (!("name" in obj)) {
    return false;
  }
  // ここで obj は object & { name: unknown } 型となる
  return typeof obj.name === "string";
}
```

この例の`checkName`は、渡された値が文字列型の`name`プロパティを持つオブジェクトかどうか調べる関数です。最終的に`typeof obj.name === "string"`を確かめるのがやりたいことですが、いきなり`obj.name`にアクセスしようとするとコンパイルエラーになります。

最初のif文で`obj`を絞り込むことで`obj`が`object`型になります。in演算子の右オペランドはオブジェクトでなければいけないため、このような絞り込みをしています。

次のif文でin演算子を用いた絞り込みを行っており、`obj`が`name`プロパティを持つことが確認できます。ただ、`name`の中身が分からないので`unknown`型として扱われます。よって、`obj`が`{ name: unknown }`型に絞り込みまれます（従来の`object`型の情報を消さないためにインターセクション型を用いて`object & { name: unknown }`と表されます）。ここがTypeScript 4.9の新機能です。

この機能は、上の例のようにユーザー定義型ガードを書く際に便利です。また、公式の例では`package.json`の読み込みが例に挙げられていました。外部のJSONファイルを読み込む場合は、何が書いてあるのか分からないため型を`unknown`型とせざるを得ません。そのような値に対して探索的にデータ読み込みを行う場合にはin演算子が便利です。

ただし、一点注意があります。それは、このようなin演算子の利用が、ランタイムエラーを防ぐという意味での型安全性に直接的に寄与しているわけではないという点です。というのも、`obj`がnullまたはundefinedではないと分かった時点で、ランタイムでは`obj`に好きな名前でプロパティアクセスしてもエラーにならないはずだからです。そのため、型安全性のためだけなら次のような方法でも代替できます。こちらが従来筆者が好んでいた方法です。

```ts
function checkName(obj: unknown): obj is { name: string } {
  if (obj === null || typeof obj !== "object") {
    return false;
  }
  const obj2: Partial<Record<string, unknown>> = obj;
  return typeof obj2.name === "string";
}
```

従来の方法と比べると、in演算子は特定のプロパティ名のみをピンポイントで拡張できるため、タイプミスにより強いという利点があるかもしれません。

