---
title: "JSXランタイムの改良: JSXオブジェクトとrender関数"
---

前章までで作ったJSXランタイムは単純なもので、`jsx`関数の結果（すなわち`<div>...</div>`のようなJSX式の結果）が直接文字列になっていました。しかし、これではあまり応用がききません。

この章では、Reactに近い仕組みに改良します。つまり、JSX式では式の内容を表すオブジェクトを組み立てるだけにして、それを`render`関数で文字列に変換するようにします。

## JSXランタイムの修正

ということで、まず`jsx`関数は次のように直します。

```ts:src/my-jsx/jsx-runtime.ts （前半）
export interface MyJSXElement {
  tag: string | ((props: Record<string, unknown>) => MyJSXElement);
  props: Record<string, unknown>;
}

export function jsx(
  tag: string | ((props: Record<string, unknown>) => MyJSXElement),
  props: Record<string, unknown>,
): MyJSXElement {
  return { tag, props };
}

export { jsx as jsxs };
```

`jsx`関数はただ`MyJSXElement`オブジェクトにデータを詰めるだけにしました。つまり、JSX式の結果はこの`MyJSXElement`になります。

そして、従来あった文字列への変換ロジックは`render`関数に移しました。

```ts:src/my-jsx/jsx-runtime.ts （続き）
export type Renderable =
  | string
  | MyJSXElement
  | Renderable[]
  | null
  | undefined;

export function render(renderable: Renderable): string {
  if (Array.isArray(renderable)) {
    return renderable.map(render).join("");
  }
  if (typeof renderable === "string") {
    return renderable;
  }
  if (renderable === undefined || renderable === null) {
    return "";
  }
  const { tag, props } = renderable;

  if (typeof tag === "function") {
    return render(tag(props));
  }

  const { children, ...rest } = props;
  const attributes = Object.entries(rest)
    .map(([key, value]) => ` ${key}="${value}"`)
    .join("");
  const innerHTML = render(children as Renderable);
  return `<${tag}${attributes}>${innerHTML}</${tag}>`;
}
```

新たに`Renderable`型を定義しました。JSX式はある種の木構造を取り、その末端は文字列になります。よって、`render`は再帰関数で定義するのが自然であり、このように`Renderable`型を定義して再帰の対象とすることできれいに書くことができます。子要素が無い場合に備えて`null`と`undefined`にも対応しました。

`Renderable`は、Reactで言うところの`ReactNode`に当たるものです。React関連の型に馴染みのある方はこの説明がわかりやすいかもしれません。

1箇所`as`が出てきてしまっていますが、この後紹介するJSXの型定義で辻褄を合わせます。

## JSXの型定義

JSXランタイムが修正されたのに合わせて、型定義も修正します。

```ts:src/my-jsx/types.d.ts
import type { MyJSXElement, Renderable } from "./jsx-runtime";

interface HasChildren {
  children?: Renderable;
}

declare global {
  namespace JSX {
    interface IntrinsicElements {
      div: HasChildren;
      h1: HasChildren & { id?: string };
      p: HasChildren;
    }

    type Element = MyJSXElement;

    interface ElementChildrenAttribute {
      children: unknown;
    }
  }
}
```

このように`JSX.Element`型を`MyJSXElement`にすることで、ランタイムの挙動と合致します。

`HasChildren`では`children`を`Renderable`に変更しました。これはJSXタグの中身は入れ子になったJSX式でもいいし、文字列でもいいということを表しています。

## 使ってみる

新しくなったJSXランタイムはこのように使います。

```tsx:src/index.tsx
import { render } from "#my-jsx/jsx-runtime";

const element = (
  <div>
    <h1 id="hello">Hello, world!</h1>
    <p>Goodbye, world!</p>
  </div>
);

console.log(render(element));
```

こうすることで、まずJSX式の結果（`MyJSXElement`）が変数`element`に入り、それを`render`関数で文字列に変換しています。表示されるのはこれまでと同じHTML文字列です。

ちなみに、今回は`render`関数を`#my-jsx/jsx-runtime`からインポートするようにしていますが、ファイルを分けて`render`関数を別のファイルで定義しても問題ありません。

## Fragmentの定義

そういえば、JSXランタイムは`Fragment`もエクスポートしていました。新しいJSXランタイムに合わせると`Fragment`はこのように定義すれば動きます。

```ts:src/my-jsx/jsx-runtime.ts （続き）
export function Fragment({ children }: { children: Renderable }): Renderable {
  return children;
}
```

ということで、この章ではJSXランタイムを改良し、応用範囲を広げました。次章ではこれを活かして、コンポーネントの機能を拡張してみましょう。