---
title: "令和5年に知っているべきTypeScriptのnamespaceの知識"
emoji: "📛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

TypeScriptには`namespace`という構文が存在します。この構文はTypeScript初期からある独自構文の一つですが、現在では特殊な用途以外では使う理由が無いため、よく知らないという方も多いでしょう。

実際、**一部のレアケースを除いて`namespace`を使う必要はありません**が、それでも知識としてあったほうが良いことが多少あります。この記事ではこの部分を解説します。

## 型に`.`でアクセスできるやつ

TypeScriptを使っていると`.`を使って型にアクセスする機会があるでしょう。例えば`React.FC`などです。

```ts
export const Page: React.FC = () => // ...
```

実は、`親.型名`のように`.`を使って型にアクセスできるのは、namespaceの機能です。上のコードでの`React`は単なる型や単なる変数ではなくnamespaceなのです。

試しに、`Foo.Bar`が`string`となるように`Foo`を定義してみてください。これができる方法は2つしかありません。そのうちの1つが`namespace`です。次のように`Foo`を定義すればできます。

```ts
namespace Foo {
  export type Bar = string;
}
```

このように関連する型をまとめて提供したい場合は`namespace`を使うことができます。この構文の中では、まるでモジュールのように`export`を使うことができます（実際、`namespace`は昔は`module`という構文でした）。

:::message
`namespace`から型ではなく変数をエクスポートすることでランタイムの挙動を持たせることもできますが、マジで使わないので今回は紹介しません。
:::

## namespaceを発生させるもう一つの方法

もう一つ、`namespace`構文を使わずに`Foo.Bar`を定義する方法があります。それは`import *`構文を使用することです。

```ts:Foo.ts
export type Bar = string;
```

```ts
import type * as Foo from "./Foo";

// これでFoo.Barがstringとなる
```

ECMAScript仕様では`import *`構文によって作られる変数（上の例の`Foo`）をモジュール名前空間オブジェクト (_module namespace object_) と呼びます。この変数はTypeScriptの型システム上ではnamespaceとして扱われるので、`Foo.Bar`のように`.`で中の型にアクセスできます。

こちらを前提に置くと、`namespace`というのは1つのファイルの中でネストしたモジュールを定義するものに見えてきます。実際、現在[ECMAScriptにもネストしたモジュール定義を導入したいという動き](https://github.com/tc39/proposal-module-declarations)もあるので、TypeScriptは期せずして10年くらい時代を先取りしていたことになりますね。

## まとめ

TypeScriptでnamespaceを発生させる方法は、明示的に`namespace`構文を使う方法と、`import *`構文を使う方法があります。

namespaceは、`Foo.Bar`のように`.`でアクセスできる型を提供する場合に有用です。他の用途では使わなくていいです。