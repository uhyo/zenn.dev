---
title: "コンポーネントのpropsの型を拡張する"
---

JSXの型定義について、まだ解説していないことが1つありました。本書の最後のトピックとしてこれを解説します。

これは、**あらゆる関数コンポーネントが受け付けるprops**の定義する方法です。

## あらゆる関数コンポーネントが受け付けるprops

これまで見たように、関数コンポーネント（前章までの例の`Title`など）がJSX式において受け取るpropsは、その関数コンポーネントの引数の型によって決まります。例えば、`Title`が`text`という名前の`string`型の引数を受け取るなら、JSX式`<Title text="Hello, world!" />`は`text`という名前の`string`型のpropsを受け取ることになります。

しかし、これだけでは表現できないユースケースがあります。Reactの使用経験がある型は、`key`についてご存じでしょう。これは、どんなコンポーネントに対してもpropsの一つとして渡すことができ、関数の引数の型に明記されていなかったとしても渡すことができます。これは前章までの説明では解決できません。

## `JSX.LibraryManagedAttributes`

このようなケースに対応するために、TypeScriptは`JSX.LibraryManagedAttributes`という機能を提供しています。お察しのとおり、`JSX`名前空間に定義することで利用できます。

この機能は名前のとおり、「ライブラリが管理する属性」を定義するためのものです。これを使うと関数コンポーネントに渡すのではなく、ライブラリ（JSXランタイム、つまりこの本の例で言えば`render`）が処理するpropsの型を定義できます。

例として、どんなコンポーネントでも`repeat`という名前の`number`型のpropsを受け付けるようにするとします。`JSX.LibraryManagedAttributes`を使うと、次のように書くことができます。

```tsx:src/my-jsx/types.d.ts（抜粋）
declare namespace JSX {
  // ...
  type LibraryManagedAttributes<_F, P> = P & {
    repeat?: number;
  }
}
```

このようにすると、`repeat`という名前の`number`型のpropsを受け付けるようになります。`_F`は関数コンポーネントの型を表しますが、今回は使っていません。`P`はpropsの型を表します。

この場合、関数コンポーネントは本来のpropsの型（`P`）に加えて`repeat`という名前の`number`型のpropsを受け取ることになります。

## ランタイムを実装する

一応ランタイムも実装しておきましょう。`render`関数に`repeat`の対応を追加します。指定されていたら、その回数だけ出力が繰り返されることにしましょう。

```ts:src/my-jsx/jsx-runtime.ts
export async function render(renderable: Renderable): Promise<string> {
  if (Array.isArray(renderable)) {
    const renderedChildren = await Promise.all(renderable.map(render));
    return renderedChildren.join("");
  }
  if (typeof renderable === "string") {
    return renderable;
  }
  if (renderable === undefined || renderable === null) {
    return "";
  }
  const { tag, props } = renderable;
  const repeat = typeof props.repeat === "number" ? props.repeat : 1;

  if (typeof tag === "function") {
    return (await render(await tag(props))).repeat(repeat);
  }

  const { children, ...rest } = props;
  const attributes = Object.entries(rest)
    .map(([key, value]) => ` ${key}="${value}"`)
    .join("");
  const innerHTML = await render(children as Renderable);
  return `<${tag}${attributes}>${innerHTML}</${tag}>`.repeat(repeat);
}
```

## 使ってみる

以上の実装で、例えば次のようなコードが型チェックを通ります。

```tsx:src/index.tsx
import { render } from "#my-jsx/jsx-runtime";

const Title = (props: { text: string }) => {
  return <h1>{props.text}</h1>;
};

const element = (
  <div>
    <Title text="Hello, world!" repeat={3} />
    <p>Goodbye, world!</p>
  </div>
);

console.log(await render(element));
```

実行結果はこうなります。

```sh
$ node dist/index.js
<div><h1>Hello, world!</h1><h1>Hello, world!</h1><h1>Hello, world!</h1><p>Goodbye, world!</p></div>
```

## 組み込み要素に対する型定義

実は上の実装にはすこし齟齬があります。それは、組み込み要素に`repeat` propを指定した場合です。

今の実装では、ランタイムは`repeat`に対応していますが、型定義が対応していません。次のようなコードがエラーとなってしまいます。

```tsx:src/index.tsx
const element = (
  <div repeat={3}>
    <Title text="Hello, world!" />
    <p>Goodbye, world!</p>
  </div>
);
```

エラー内容:

```
src/index.tsx:8:8 - error TS2322: Type '{ children: MyJSXElement[]; repeat: number; }' is not assignable to type 'HasChildren'.
  Property 'repeat' does not exist on type 'HasChildren'.

8   <div repeat={3}>
```

これは、`JSX.LibraryManagedAttributes`はあくまで関数コンポーネントに対して適用されるもので[^note_class_component]、組み込み要素には適用されないからです。そのため、組み込み要素の型定義は別途行う必要があります。

[^note_class_component]: 正確にはクラスコンポーネントにも適用されますが、本書ではクラスコンポーネントは扱っていません。

組みこみ要素にも対応するように、DRYを意識しつつ型定義を改良するとこんな感じになるでしょう。

```tsx:src/my-jsx/types.d.ts
import type {
  MyJSXElement,
  Renderable,
  FunctionComponentResult,
} from "./jsx-runtime";

interface PropsForAnyJSXElement {
  repeat?: number;
}

interface PropsForAnyIntrinsicElement extends PropsForAnyJSXElement {
  children?: Renderable;
}

declare global {
  namespace JSX {
    interface IntrinsicElements {
      div: PropsForAnyIntrinsicElement;
      h1: PropsForAnyIntrinsicElement & { id?: string };
      p: PropsForAnyIntrinsicElement;
    }

    type Element = MyJSXElement;
    // 追加
    type ElementType = string | ((props: any) => FunctionComponentResult);

    interface ElementChildrenAttribute {
      children: unknown;
    }

    type LibraryManagedAttributes<_F, P> = P & PropsForAnyIntrinsicElement;
  }
}
```

これで、組み込み要素にも`repeat` propを指定できるようになります。実際のReactの型定義なども、委細は異なりますが、おおよそこんな感じです。

## `JSX.LibraryManagedAttributes`の他の例

上の例では`_F`を使っていなかったので、使う例も（型定義だけですが）紹介しておきます。

例えば、非同期コンポーネント（Promiseを返す）のときに限り`timeout` propを指定できるようにしたければ、次のように書くことができます。

```tsx
type LibraryManagedAttributes<F, P> = F extends () => Promise<any>
  ? P & { timeout?: number }
  : P;
```

`F`が関数自体の型であることに注意してください。

他にも、Reactでは`defaultProps`という機能がありました（[関数コンポーネントの`defaultProps`はReact 19で廃止されますが](https://zenn.dev/uhyo/books/react-19-new/viewer/upgrade)）。

これは関数オブジェクト自体に`defaultProps`というプロパティを持たせることで効果を発揮するものであるため、定義の仕方としては次のようになります。

```tsx
type LibraryManagedAttributes<F, P> = F extends { defaultProps: infer DP }
  ? P & /* 省略 */
  : P;
```

このように、`JSX.LibraryManagedAttributes`は応用の幅が広く様々な用途に使えることがわかります。これがこの本でお伝えする最後のトピックです。

ここまでの知識があれば、自由にJSXランタイムを実装することができるはずです。この本ではHTML文字列の生成という大したことのない例でしたが、アイデア次第で様々なことができるでしょう。