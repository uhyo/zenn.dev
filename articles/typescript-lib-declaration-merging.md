---
title: "TypeScriptの標準ライブラリで使われているdeclaration mergingのテクニック"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

TypeScriptの標準ライブラリとは、TypeScriptに組み込みで備わっている型定義のことです。ECMAScript仕様で定義されているJavaScriptの言語機能に対する型定義が含まれています。また、ブラウザに組み込まれているWeb標準のAPIに対する型定義も含まれています。

TypeScriptの標準ライブラリでは、**declaration merging**というテクニックが使われています。皆さんが普段書くアプリケーションコードではあまり使う機会がないかもしれませんが、TypeScriptの型定義、とりわけ標準ライブラリの型定義においては重要なテクニックです。

この記事では、declaration mergingの概要と、TypeScriptの標準ライブラリでどのように使われているかについて解説します。

## declaration mergingとは

Declaration mergingとは、同じ名前のinterfaceを別々の場所で複数回定義した場合、それらがマージされて1つのinterfaceとして扱われるという仕組みです。これにより、型定義を分割して記述することができます。

```ts
interface User {
  name: string;
}

interface User {
  age: number;
}

function greet(user: User) {
  // userは{name: string; age: number;}という型になる
  console.log(`Hello, ${user.name}! You are ${user.age} years old.`);
}
```

この例では、`User`という名前のinterfaceを2回定義しています。TypeScriptはこれらをマージして1つのinterfaceとして扱います。そのため、`User`型は`{name: string; age: number;}`という型になります。

よくtypeとinterfaceが比較されますが、typeは同じ型名を複数回定義するとエラーになりますが、interfaceはdeclaration mergingによりマージされるため、同じ名前のinterfaceを複数回定義することができるという違いがあります。

## TypeScriptの標準ライブラリでのdeclaration merging

TypeScriptの標準ライブラリでは、interfaceが多用されています。`type`よりも`interface`のほうが型チェックのパフォーマンスが良いという説もありそれも理由のひとつかもしれませんが、やはりdeclaration mergingができることが大きな理由です。

ポイントは、TypeScriptの標準ライブラリはESバージョンごとに型定義を分割していることです。たとえば、`lib.es5.d.ts`にはES5の標準ライブラリの型定義が含まれています。同様に、`lib.es2015.d.ts`にはES2015の標準ライブラリの型定義が含まれています。TypeScriptでは`target`オプションまたは`lib`オプションで使用する型定義を指定することができ、例えば`es2023`を指定するとes5, es2015, …, es2023の型定義が読み込まれます。

ECMAScriptバージョンが上がると、既存のオブジェクトにメソッドが追加されることがあります。たとえば、ES2016で`Array.prototype.includes`が追加されました。このことをうまく表現するためにdeclaration mergingが使われています。

配列を表す`Array`型は昔から存在するため、ES5の型定義で`Array`型が定義されています。`lib.es5.d.ts`には、ES5ですでに存在したメソッドだけが`Array`のメソッドとして定義されています。

```ts
interface Array<T> {
  // ES5のArray型のメソッド
  pop(): T | undefined;
  push(...items: T[]): number;
  // ...
}
```

ES2016で`Array.prototype.includes`が追加されたため、`lib.es2016.d.ts`には`Array`型のメソッドとして`includes`が追加されています[^es2016]。

[^es2016]: 正確には、`lib.es2016.array.include.d.ts`に定義されており、`lib.es2016.d.ts`は個別の定義ファイルをとりまとめる役割を持っています。

```ts
interface Array<T> {
  includes(searchElement: T, fromIndex?: number): boolean;
}
```

`target: "es2016"`の状況では、ES5, ES2015, ES2016の型定義が全て読み込まれるため、複数の`Array<T>`の定義がマージされます。これにより、実際の`Array`型はES2016までのメソッドを全て持つ型となります。

このように、ESバージョンが上がるにつれて段階的に型定義を追加していくことができるのは、declaration mergingがあるからこそです。このため、declaration mergingはTypeScriptに欠かせない機能となっています。

## declaration mergingの応用

Declaration mergingは、単に新しいメソッドを追加する以外の使い道もあります。標準ライブラリでもそのような応用的な使い方が見られる場面もあります。

`Intl.NumberFormat`は、数値のフォーマットを行うAPIです。ロケール（言語）に応じた適切な桁区切り文字の付与や、通貨単位の表示などが可能です。

```ts
const numberFormat = new Intl.NumberFormat('ja-JP', {
  style: 'currency',
  currency: 'JPY',
});

console.log(numberFormat.format(1234567)); // => "￥1,234,567"
```

上の例では`style`オプションに`'currency'`を指定しています。今のところ、`style`に指定できる値は4つで、`'currency'`以外に`'decimal'`, `'percent'`, `'unit'`があります。

このうち、`'unit'`はES2020で追加されたものです[^ecma402]。このように、Intl APIでは既存のメソッド等のオプションが追加されることが良くあります。

[^ecma402]: `Intl`はECMAScriptの仕様 (ECMA-262) とは別の仕様 (ECMA-402) で定義されているためES2020と呼んでいいのかは微妙ですが、とにかくECMA-402の2020年版には追加されています。

TypeScriptでは型定義上でこの仕様を表現することに成功しています。`style`に`'unit'`を渡せるのは、ES2020以降の型定義が読み込まれている場合のみです。

```ts
const numberFormat = new Intl.NumberFormat('ja-JP', {
  style: 'unit',
  //     ^^^^^^ ES2019以前では型エラーが発生！
  unit: 'meter',
});

console.log(numberFormat.format(1234567)); // => "1,234,567 m"
```

これはどのように実現されているのでしょうか。単純にdeclaration mergingで新しい関数定義を追加するというわけにもいきません。

実は、ここでdeclaration mergingの賢い応用がされています。関連する型定義を`lib.es5.d.ts`から抜粋します。

```ts:lib.es5.d.ts
interface NumberFormatOptionsStyleRegistry {
  decimal: never;
  percent: never;
  currency: never;
}

type NumberFormatOptionsStyle = keyof NumberFormatOptionsStyleRegistry;

interface NumberFormatOptions {
  style?: NumberFormatOptionsStyle | undefined;
  // ...
}
```

この型定義単体では、`NumberFormatOptionsStyle`は`'decimal' | 'percent' | 'currency'`というユニオン型になります。つまり、`'unit'`を指定できないことがわかります。

一方、ES2020の型定義では、次の型定義が書かれています。

```ts:lib.es2020.intl.d.ts
interface NumberFormatOptionsStyleRegistry {
  unit: never;
}
```

ここでdeclaration mergingが行われることで、`keyof NumberFormatOptionsStyleRegistry`の結果が変わり、`'decimal' | 'percent' | 'currency' | 'unit'`というユニオン型になります。これにより、ES2020の型定義を読み込めば`style`に`'unit'`を指定できるようになります。

このように、`keyof`を使うことで、declaration mergingを文字列のユニオン型の拡張に使うこともできるのです。このテクニックはIntlの型定義でよく見られます。

この目的で定義されるinterfaceには`Registry`という名前が付けられるので、筆者は個人的にこの型定義の書き方を「Registryパターン」と呼んでいます。

## 標準ライブラリ以外の例

標準ライブラリ以外でも、declaration mergingが使われてることがあります。たとえば、Reactの型定義 (`@types/react`) を見てみましょう。

この記事の公開時点ではReactの最新メジャーバージョンは18であり、次のバージョンである19はReact 19 RCが出ているという状況です。そのため、`@types/react`はReact 18向けの型定義となっています。しかし、`@types/react`はReact 19向けの型定義もcanaryとして提供されています。次のようにすることで、React 19向けの型定義を読み込むことができます。

```ts
import type {} from 'react/canary';
```

この状態では、メインの`@types/react`と`react/canary`が両方読み込まれることで、React 18向けの型定義がReact 19向けにアップグレードされます。2つの型定義を両方読み込むということで、基本的にはdeclaration mergingが活用されていると考えられます。

React 19では、Reactのノードとして扱われる値の種類が増えます。特に、Server Componentsではasync関数をReactコンポーネントとして扱えるようになります。これは言い換えれば、React 19では関数コンポーネントの返り値として、従来の値に比べてPromiseも許されるということです。具体的には、関数コンポーネントの返り値として許されるのは`ReactNode`型であり、[次のように定義されています](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/e024c092d7447d5b6e630df798574ed5427cae19/types/react/index.d.ts#L446)。

```ts
/**
 * Different release channels declare additional types of ReactNode this particular release channel accepts.
 * App or library types should never augment this interface.
 */
interface DO_NOT_USE_OR_YOU_WILL_BE_FIRED_EXPERIMENTAL_REACT_NODES {}

type ReactNode =
    | ReactElement
    | string
    | number
    | Iterable<ReactNode>
    | ReactPortal
    | boolean
    | null
    | undefined
    | DO_NOT_USE_OR_YOU_WILL_BE_FIRED_EXPERIMENTAL_REACT_NODES[
        keyof DO_NOT_USE_OR_YOU_WILL_BE_FIRED_EXPERIMENTAL_REACT_NODES
    ];
```

お分かりのように、ごつい名前のinterfaceが用意されています。ここでRegistryパターンが使われています。名前は有名な`SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED`のもじりでしょう。

この定義により、`ReactNode`のユニオン型に別のファイルから新しい型を追加することができます。`react/canary`では[次のように定義されています](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/e024c092d7447d5b6e630df798574ed5427cae19/types/react/canary.d.ts#L153C1-L156C6)。

```ts
interface DO_NOT_USE_OR_YOU_WILL_BE_FIRED_EXPERIMENTAL_REACT_NODES {
    promises: Promise<AwaitedReactNode>;
    bigints: bigint;
}
```

これにより、`react/canary`を読み込んだ場合、`ReactNode`に`Promise<AwaitedReactNode>`と`bigint`が追加されます。

このパターンは標準ライブラリで見られたパターンをさらに応用したものであり、文字列のユニオン型にとどまらず任意の型をユニオン型に追加することができます。TypeScriptの強力な型計算機能と組み合わせることで、Registryパターンは自由自在に型定義を拡張することができると言えます。

## まとめ

この記事では、TypeScriptの標準ライブラリやReactの型定義などで使われている、declaration mergingとその応用について解説しました。

複数の型定義ファイルが協調して1つの型定義を構築するようなケースでは有効なテクニックです。
