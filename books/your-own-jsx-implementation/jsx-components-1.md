---
title: "コンポーネント定義をサポートする"
---

JSXの強力なところは、**コンポーネントを定義できる**ことです。Reactでは関数をコンポーネント（関数コンポーネント）として使うことができますし、その前にあったクラスコンポーネントも今でも使えます。

ということで、この章では自前のJSXランタイムにコンポーネントの機能を追加してみましょう。

TypeScriptは、Reactのような**関数コンポーネント**をサポートしています。つまり、コンポーネントは「引数でpropsを受け取ってJSXを返す関数」として定義します。

```tsx:src/index.tsx
const Title = (props: { children: string }) => <h1>{props.children}</h1>;

const element = <div>
  <Title>Hello, world!</Title>
  <p>Goodbye, world!</p>
</div>;

console.log(element);
```

これはまだ実行結果もおかしいし型エラーも出るので、これから直していきましょう。

## ランタイムを直す

上記のコードを現状のまま実行すると、こんな出力になってしまいます。

```
<div><(props) => _jsx("h1", { children: props.children })>Hello, world!</(props) => _jsx("h1", { children: props.children })><p>Goodbye, world!</p></div>
```

これは、我々が用意した`jsx`関数が関数コンポーネントに対応していないからです。この場合にトランスパイル後のコードがどうなっているのかも見てみましょう。

```js:dist/index.js
import { jsx as _jsx, jsxs as _jsxs } from "#my-jsx/jsx-runtime";
const Title = (props) => _jsx("h1", { children: props.children });
const element = _jsxs("div", {
  children: [
    _jsx(Title, { children: "Hello, world!" }),
    _jsx("p", { children: "Goodbye, world!" }),
  ],
});
console.log(element);
```

なるほど、`<Title>...</Title>`のようなコンポーネントの使用は、`jsx`関数の第一引数に文字列ではなくコンポーネントがそのまま渡されることで表現されるようです。

ということで、JSXランタイムを修正してコンポーネントをサポートするようにしましょう。

```ts:src/my-jsx/jsx-runtime.ts
export function jsx(
  tag: string | ((props: Record<string, unknown>) => JSX.Element),
  props: Record<string, unknown>,
): string {
  if (typeof tag === "function") {
    return tag(props);
  }
  const { children, ...rest } = props;
  const attributes = Object.entries(rest)
    .map(([key, value]) => ` ${key}="${value}"`)
    .join("");
  const innerHTML = Array.isArray(children) ? children.join("") : children;
  return `<${tag}${attributes}>${innerHTML}</${tag}>`;
}

export { jsx as jsxs };
```

これは普通に`tag`が関数の場合の考慮を追加すればいいだけです。細かい型の帳尻合わせは型定義のほうでやればよいでしょう。

このようにランタイムを修正して実行すれば、期待通りの結果が得られます。

```
<div><h1>Hello, world!</h1><p>Goodbye, world!</p></div>
```

## 型を直す

次に、型エラーを直していきましょう。現状、`src/index.tsx`に対してはこんな型エラーが出ているはずです。

```
src/index.tsx:5:6 - error TS2745: This JSX tag's 'children' prop expects type 'string' which requires multiple children, but only a single child was provided.

5     <Title>Hello, world!</Title>
       ~~~~~
```

実はこの型エラーはちょっと言っていることがおかしいので、TypeScriptにバグ報告を出しておきました。

https://github.com/microsoft/TypeScript/issues/60572

とりあえず、このエラーを解消するためにはJSXの型定義に次のように`ElementChildrenAttribute`を追加すればよいです。

```tsx:src/my-jsx/types.d.ts
interface HasChildren {
  children?: string | string[];
}

declare namespace JSX {
  interface IntrinsicElements {
    div: HasChildren;
    h1: HasChildren & { id?: string };
    p: HasChildren;
  }

  type Element = string;

  interface ElementChildrenAttribute {
    children: unknown;
  }
}
```

これは簡単に言えば「子要素を表すprops名は`children`だよ」という宣言になります（`unknown`部分は何でもよくて、`children`という名前を指示しているところに意味があります）。本当は`"jsx": "react-jsx"`を使っているときには不要だと思うのですが、現在のTypeScriptの挙動ではこれを追加しないとエラーが出るようです。

ということで、これを追加すれば無事に型エラーが消えました。

次の章では、JSXの機能であるフラグメント記法に対応します。