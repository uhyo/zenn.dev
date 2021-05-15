---
title: "TypeScript 4.2 と 4.3 で起こった“最弱の更新”"
emoji: "👾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

TypeScript 4.3 の RC が先日登場し、正式リリースが近づいてきていますね。そこで、この記事では TypeScript 4.2 から 4.3 にかけて発生した“最弱の更新”とも言える出来事、そしてそれによって起こったベストプラクティスの変化を紹介します。

## TypeScript 4.2 でコンストラクトシグネチャの abstract 修飾子の追加

コンストラクトシグネチャ (construct signature) とは、コンストラクタ、すなわち「`new`できる関数」を表す型システム上の概念です。[TypeScript 4.2 からは、コンストラクトシグネチャに`abstract`修飾子を付けることができるようになりました](https://devblogs.microsoft.com/typescript/announcing-typescript-4-2/#abstract-construct-signatures)。

```ts
// 普通のコンストラクトシグネチャを持つ関数オブジェクト
declare const foo: new (arg: number) => unknown;

const obj1: unknown = new foo(123);

// abstractコンストラクトシグネチャを持つ関数オブジェクト
declare const bar: abstract new (arg: number) => unknown;

// これはエラーになる
// エラー: Cannot create an instance of an abstract class.
const obj2: unknown = new bar(123);
```

従来も TypeScript には abstract クラスという概念があり、abstract クラスを`new`できない機能はすでにありました。しかし、この abstract 性は具体的なクラス型（`abstract class`構文によってのみ得られる型）にのみ与えられるフラグだったのです。

```ts
// abstractクラスの宣言
abstract class Abs {}

// エラー: Cannot create an instance of an abstract class.
const obj1: unknown = new Abs();
```

従来から、abstract クラスは「クラスを受け取る関数」に渡すことができませんでした。

```ts
// エラー: Argument of type 'typeof Abs' is not assignable to parameter of type 'new () => unknown'.
//          Cannot assign an abstract constructor type to a non-abstract constructor type.
useClass(Abs);

function useClass(cls: new () => unknown) {
  new cls();
}
```

つまり、従来は「abstract クラスでも受け取ることができる関数」を宣言する方法が欠如していました。これを可能にしたのが TypeScript 4.2 の abstract コンストラクトシグネチャです。

```ts
// abstractクラスを渡せる！
useClass(Abs);

function useClass(cls: abstract new () => unknown) {}
```

これにより、abstract クラスという概念が型システム上で一級市民としての地位を得たことになります。これが TypeScript 4.2 で起こった変更です。

## TypeScript 4.3 での標準ライブラリの変更

TypeScript 4.3 では、[abstract コンストラクトシグネチャが標準ライブラリに取り入れられました](https://github.com/microsoft/TypeScript/pull/43380)。

```diff ts
- type InstanceType<T extends new (...args: any) => any> = T extends new (...args: any) => infer R ? R : any;
+ type InstanceType<T extends abstract new (...args: any) => any> = T extends abstract new (...args: any) => infer R ? R : any;
```

これによって、従来は不可能だった「`InstanceType`型に abstract クラス（の型）を渡す」といったことが可能になりました。

## abstract コンストラクトシグネチャによって起こった“最弱の更新”

上記のような標準ライブラリの更新が、abstract コンストラクトシグネチャによって起こった **“最弱の更新”** を表しています。最弱というのは、この記事では型の強さ弱さを指しています。つまり、強い型というのは制限の強い型で弱い型というのは制限の弱い型です。TypeScript において最も弱い型は、一切の制限なくどんな値でも受け入れる`unknown`型です。

関数を定義する場合、引数の型は可能な範囲で最弱にするのが望ましいと考えられます。そのほうがより汎用的に使える関数になるからからです。abstract コンストラクトシグネチャの場合、「コンストラクタを受け取るが自分で`new`しない関数」では、従来は`new () => unknown`のような型を受け取るのが最弱の選択肢であったところ、新たに`abstract new () => unknown`型の登場によって最弱が更新されました。「コンストラクタを受け取るが自分で`new`しない関数」の具体例としては、TypeScript 公式のリリースノートにもあるような、与えられたコンストラクタを`extends`するためだけに使う関数が挙げられます（次の例は TypeScript の推論力が足りなくて中身が`as`まみれになっていますが、インターフェースに注目してください）。

```ts
// 従来の最弱
function addDefaultParam<T extends object>(cls: new (str: string) => T) {
  class A extends (cls as new (str: string) => object) {
    constructor() {
      super("pikachu");
    }
  }
  return A as new () => T;
}

// 新たな最弱
function addDefaultParam2<T extends object>(
  cls: abstract new (str: string) => T
) {
  class A extends (cls as abstract new (str: string) => object) {
    constructor() {
      super("pikachu");
    }
  }
  return A as new () => T;
}
```

また、TypeScript 4.3 のリリースノートにあった`InstanceType`のような例も、型上で“最弱の更新”が起こった例であると見ることができます。

最弱の更新というのは、常識の更新でもあります。これまでは「（関数内部で new されない）クラスを受け取る関数を汎用的にしたければコンストラクトシグネチャを受け取る型を書けばベスト」という常識があったところ、TypeScript 4.2 の機能追加によってその常識は更新され、「（関数内部で new されない）クラスを受け取る関数を汎用的にしたければ**abstract**コンストラクトシグネチャを受け取る型を書けばベスト」となりました。TypeScript 4.3 の標準ライブラリの変化はこの新しい常識に対応した結果です。

このように、TypeScript の機能追加によって常識の更新が起こることがあります。TypeScript 使いの皆さんは、ぜひこのような変化を見逃さずに最新の常識に基づいてコードを書けるようにしておきましょう。

## “最弱の更新”の他の例

TypeScript の歴史の中で“最弱の更新”が起こった例は他にもあります。顕著なのは**readonly 配列型**です。TypeScript 2.0 で`ReadonlyArray<T>`型が追加され、TypeScript 3.4 では`readonly T[]`という構文も追加されました。従来「配列を受け取るが破壊的変更をしない関数」の引数の型のベストプラクティスは`T[]`型でしたが、readonly 配列型の追加によって`readonly T[]`型に変わりました。今や、引数の型を`readonly T[]`ではなくわざわざ`T[]`にすることは、「この関数は与えられた配列を破壊的に変更します」という意思表示であるといっても過言ではありません。これもやはり、TypeScript の機能追加によって常識が変化し、その結果プログラムから読み取られる意図までもが変化してしまった例です。

## まとめ

TypeScript では、型システムの進化に伴って“最弱の更新”という形でベストプラクティスが変化することがあります。TypeScript 4.2 以降では、クラスを受け取るが`new`はしない関数の場合は引数を abstract コンストラクトシグネチャ型とするのが新たなベストプラクティスとなりました。
