---
title: "Biome v2の型推論を試して限界を知る"
emoji: "🌳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "biome"]
published: true
---

皆さんこんにちは。先日、**Biome v2**がリリースされ話題となりました。Biome v2の新機能の一つに**型推論**があります。

TypeScriptコードに対するlintにおいて型情報を使う (type-aware linting) 機能は、これまでのところTypeScript-ESLintによって提供されてきました。これは、実際のTypeScriptコンパイラを使って型情報を取得するので、重いという欠点がありました。TypeScript自体もGoへの移植などを通じてパフォーマンス改善に取り組んでいますが、Biomeはこの問題に対して別のアプローチをとっていました。それが、本家TypeScriptコンパイラに頼らず**独自に型推論を行う**というものです。

ただし、TypeScriptコンパイラは非常に複雑なシステムであり、別実装でその型推論結果を完全に再現するのはまず不可能です。そのため、Biomeの型推論も、完全にTypeScriptコンパイラの挙動を再現することはできません。[Biome v2のリリースノート](https://biomejs.dev/blog/biome-v2/)では、`noFloatingPromises`ルールを例にして、実際のTypeScriptコンパイラを使用した場合に比べて75%ほどのケースを検知できるとされています。

そこで、この記事では、Biome v2の型推論機能を試して、本物のTypeScriptコンパイラと比較してどの程度型推論できるのかを調べてみます。

:::message
この記事では、あくまで記事執筆時点のバージョン (2.0.4) を対象として調べています。Biomeは今後のバージョンアップで型推論を改善していくことを予告しています。
:::

## 簡単な例

今回は、`noFloatingPromises`ルールを用いて調べていきます。これは、Promiseなのにawaitされていないコードを検知するルールです。まずは、簡単な例から見ていきましょう。

```ts
async function foo() {
  return 3.14;
}

export async function main() {
  foo();
}
```

このコードは、`foo`関数がPromiseを返すのに対して、`main`関数ではその結果をawaitしていません。したがって、lintエラーが発生することが期待されます。

これに対して`biome lint`を実行すると、以下のようなエラーが出力されます。

```
index.ts:6:3 lint/nursery/noFloatingPromises  FIXABLE  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ℹ A "floating" Promise was found, meaning it is not properly handled and could lead to ignored errors or unexpected behavior.
  
    5 │ export async function main() {
  > 6 │   foo();
      │   ^^^^^^
    7 │ }
  
  ℹ This happens when a Promise is not awaited, lacks a `.catch` or `.then` rejection handler, or is not explicitly ignored using the `void` operator.
  
  ℹ Unsafe fix: Add await operator.
  
    6 │ ··await·foo();
      │   ++++++
```

`foo()`の返り値がPromiseであることを検知し、`await`が必要であると指摘していますね。`foo()`には明示的な型注釈などはありませんが、Biomeは関数の戻り値を推論してPromiseであると判断しています。

ここから、だんだん推論を難しくしていきます。

## `new Promise` を使ってみる

上の例では、`foo`が`async function`で定義されていたため、返り値がPromiseであることが非常に明らかでした。次は、`new Promise`を使ってみましょう。

```ts
function foo() {
  return new Promise<number>((resolve) => {
    setTimeout(() => {
      resolve(3.14);
    });
  });
}

export async function main() {
  foo();
}
```

この例では、関数fooの返り値がPromiseであることは変わりません。しかし、`async`キーワードを使わずに`new Promise`を使っています。

このコードに対して`biome lint`を実行すると、lintエラーは検知されませんでした。

残念ながら、この場合はBiomeは関数の返り値がPromiseであることを認識できないようです。この場合、fooの返り値の型注釈がありませんので、fooの型を調べるためにはfooの中のreturn文を探し、その値が`new Promise`であることを確認し、この式の型がPromiseであることを調べる必要があります。しかし、Biomeはこのような型推論を行うことはできないようです。

## 型注釈を明示する

では、関数の返り値に型注釈を追加してみましょう。

```ts
function foo(): Promise<number> {
  return new Promise<number>((resolve) => {
    setTimeout(() => {
      resolve(3.14);
    });
  });
}

export async function main() {
  foo();
}
```

これであれば、Biomeはlintエラーを検知できました。

Biomeの型推論の内部実装を見たわけではありませんが、関数の返り値の型を知るために関数の中身を調べるようなことはしていないのでしょう。biomeの型推論を活用するためには、関数の返り値の型を明記することが重要になります。

## Promiseを使ってみる

では、関数は分かりやすいようにasync関数に戻して、いろいろ試してみましょう。

```ts
async function foo(): Promise<number> {
  return 3.14;
}

export async function main() {
  console.log(foo());
}
```

この例に対して`biome lint`を実行すると、なんとエラーは検知されませんでした。個人的にはちょっとびっくりしました。

ただ、一応公式の説明にnoFloatingPromisesルールの挙動について以下のように書かれています。

> This rule will report Promise-valued statements that are not treated in one of the following ways:
>
> - Calling its `.then()` method with two arguments
> - Calling its `.catch()` method with one argument
> - `await`ing it
> - `return`ing it
> - `void`ing it

Promise-valued _statements_なので、Promiseを式として何かに使った場合は対象外で、あくまで`foo();`のようにPromiseを何もせず放置した場合に検知対象を限るとも解釈できます。

そのためか、以下のようにPromiseを変数に入れた場合もlintエラーは検知されませんでした。

```ts
async function foo(): Promise<number> {
  return 3.14;
}

export async function main() {
  const p = foo();
  console.log(p);
}
```

## オブジェクトを介してみる

関数ではなくオブジェクトのメソッドの場合も試してみました。

```ts
const obj = {
  foo: async () => {
    return 3.14;
  },
}

export async function main() {
  obj.foo();
}
```

この場合はlintエラーが検知されました。何らかの推論を通じて、`obj.foo`がPromiseを返す関数であることを理解したようです。

ちなみに、以下のようにするとlintエラーは消えます（代わりに`any`の使用に対するエラーが出ますが）。

```ts
const obj: {
  foo: any;
} = {
  foo: async () => {
    return 3.14;
  },
}

export async function main() {
  obj.foo();
}
```

このことから、ちゃんと「objの型」を推論し、それを介して`obj.foo`の返り値の型を推論していることが分かります。

## 難しい型を使ってみる

では、ここからは意地悪して、TypeScriptの難しい型を使ってみましょう。

### ジェネリクス

まずは、ジェネリクスを使った例です。

```ts
function id<T>(x: T): T {
  return x;
}
async function foo(): Promise<number> {
  return 3.14;
}
export async function main() {
  id(foo());
}
```

この例では、`id(foo())`の返り値がPromiseになりますが、ジェネリクスの型推論を行わないとそのことを理解できません。

筆者はここの結果が一番驚きだったのですが、なんとBiomeはこの例に対してlintエラーを検知しました。つまり、`id(foo())`の返り値がPromiseであることを認識できているようです。`foo()`ではなく`id`に対してちゃんとlintエラーが検知されています。

TypeScriptにおけるジェネリクスの推論ルールは非常に複雑なので、Biomeがその全てを再現できるとは思いませんが、この例のような単純なケースではジェネリクスを使った型推論ができるようです。

### lookup型

Lookup型とは、`T[K]`のような構文の型です。

```ts
interface Obj {
  noPromise: number;
  yesPromise: Promise<number>;
}

const obj: Obj = {
  noPromise: 3.14,
  yesPromise: Promise.resolve(3.14),
}

function foo<K extends keyof Obj>(key: K): Obj[K] {
  return obj[key];
}

export async function main() {
  foo('noPromise');
  foo('yesPromise');
}
```

こうすると、`foo('noPromise')`の返り値はnumber型ですが、`foo('yesPromise')`の返り値は`Promise<number>`型になります。Biomeはこのことを見抜けるでしょうか。

答えは、残念ながら`foo('yesPromise')`に対してlintエラーは検知されませんでした。このように、Lookup型の計算には対応していないようです。

ちなみに、ジェネリクスを交えなくても以下のような単純なケースでも無理でした。

```ts
interface Obj {
  noPromise: number;
  yesPromise: Promise<number>;
}

async function foo(): Obj["yesPromise"] {
  return 3.14;
}

export async function main() {
  foo();
}
```

### Mapped型・条件型

Lookup型が無理だった時点で残りも無理だとは思いますが、一応試してみました。やはり無理でした。

```ts
// Mapped型
type Raw = {
  noPromise: number;
  yesPromise: Promise<number>;
}

type Obj = {
  [K in keyof Raw]: () => Raw[K];
}

const obj: Obj = {
  noPromise: () => 42,
  yesPromise: () => Promise.resolve(3.14),
}

export async function main() {
  obj.yesPromise();
}
```

```ts
// 条件型
function foo<K extends string | number>(key: K): K extends string ? number : Promise<number> {
  if (typeof key === 'string') {
    return 42 as any;
  } else {
    return Promise.resolve(42) as any;
  }
}

export async function main() {
  foo(123);
}
```

## ユニオン型とインターセクション型

TypeScriptの実用において重要なのがユニオン型です。これを交えたケースを試してみましょう。

```ts
function foo(): number | Promise<number> {
  if (Math.random() > 0.5) {
    return 42;
  }
  return new Promise((resolve) => {
    setTimeout(() => resolve(42), 1000);
  });
}

export async function main() {
  foo();
}
```

この場合、`foo()`はPromiseかもしれないし、Promiseではないかもしれません。

この例に対しては、Biomeはlintエラーを検知しました。つまり、ユニオン型の一部としてPromiseの可能性があることを認識できているようです。

では、インターセクション型も試してみましょう。

```ts
function foo(): Promise<number> & { abort: () => void} {
  return {} as any; // 省略
}

export async function main() {
  foo();  
}
```

こちらは、lintエラーは検知されませんでした。インターセクション型の一部としてPromiseがあることは認識できていないようです。ただ、これを検知対象にすべきかどうかは判断が分かれるかもしれません。また、検知において本質的な難しさがあるわけではないので、比較的簡単に対応ができそうです。

## 型に別名を付けてみた

promise型に別名をつけた場合は検知されるでしょうか。

```ts
type PromiseNumber = Promise<number>;
function foo(): PromiseNumber {
  return new Promise((resolve) => {
    setTimeout(() => resolve(42), 1000);
  });
}
export async function main() {
  foo();
}
```

この例では検知できました。型の複雑な計算は無理としても、型エイリアスは認識できるようです。

## まとめ

この記事では、Biome v2の現時点での“型推論”の性能を調べてみました。

結果として、`return new Promise`の例で型推論ができず型注釈が必要になるなど、型チェッカーによる型推論と比べるとかなり控えめな性能であることが分かりました。

それでも、内部的にオブジェクト型や関数型の取り扱いがあったり、ジェネリクスを取り扱えたり、async関数の返り値は型注釈がなくてもPromise型であることを認識できたりと、推論と呼べる挙動は見られました。型推論に見せかけて別物の何かというわけではなく、“型”という概念にきちんと向き合って作られているのが感じられます。

しかし、型注釈を明記した場合でも、型の計算には限界があるようです。Lookup型や条件型などを計算することは不可能なようでした。

このような控えめな性能でも、公式ブログにあるように、75%のケースを検知するのに十分であるというのは驚きですね。

筆者は、Biomeやその他TypeScript非依存の“型推論”機能について心配なことがありました。それは、TypeScriptの型システムをフルに活用せず、Biomeなどが理解できる範囲の記述しかしないようなコーディングルールが広まってしまうのではないかということです。そうなったとき、TypeScriptの型システムは実質的にフォークすることになります。

Biomeも進化の途上にあるとはいえ、今回の検証結果を見て皆さんはどう思ったでしょうか。Biomeに合わせてTypeScriptの運用を変えていきたいと思ったでしょうか。

筆者としては、これからどうなるかまた見守っていきたいと思います。
