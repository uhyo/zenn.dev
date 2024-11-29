---
title: "フラグメント記法に対応する"
---

JSXにはフラグメント記法 `<>...</>`があります。独自のJSXランタイムでこれをサポートしてみましょう。

```tsx:src/index.tsx
const element = (
  <>
    <h1 id="hello">Hello, world!</h1>
    <p>Goodbye, world!</p>
  </>
);

console.log(element);
```

このコードは、現段階でも型チェックが通りますが、ランタイムでエラーになります。

```
import { jsx as _jsx, Fragment as _Fragment, jsxs as _jsxs } from "#my-jsx/jsx-runtime";
                      ^^^^^^^^
SyntaxError: The requested module '#my-jsx/jsx-runtime' does not provide an export named 'Fragment'
```

## ランタイムを修正する

ランタイムエラーの原因を知るために、トランスパイル後のコードを見てみましょう。

```js:dist/index.js
import {
  jsx as _jsx,
  Fragment as _Fragment,
  jsxs as _jsxs,
} from "#my-jsx/jsx-runtime";
const element = _jsxs(_Fragment, {
  children: [
    _jsx("h1", { id: "hello", children: "Hello, world!" }),
    _jsx("p", { children: "Goodbye, world!" }),
  ],
});
console.log(element);
```

すると、新たに`#my-jsx/jsx-runtime`から`Fragment`というものがインポートされていることが分かります。そして、この`Fragment`を使って、`<>...</>`は実は`<Fragment>...</Fragment>`と同じ意味になっていることが分かります。

そのため、この`Fragment`をランタイムに追加する必要があることが分かります。さっそく追加してみましょう。

```ts:src/my-jsx/jsx-runtime.ts
// 既存コードの下に追加
export function Fragment({
  children,
}: {
  children: string | string[];
}): string {
  return Array.isArray(children) ? children.join("") : children;
}
```

これで、`Fragment`が追加されました。これだけでフラグメント記法をサポートできます。とても簡単ですね。

```sh
$ npx tsc
$ node dist/index.js
<h1 id="hello">Hello, world!</h1><p>Goodbye, world!</p>
```

## Fragmentの名前を変えたい場合

実は、`<>`記法に対応するコンポーネントの名前はデフォルトで`Fragment`ですが、コンパイラオプションで変更できます。そのためには、`jsxFragmentFactory`というオプションを使います。

```tsconfig.json
{
  "compilerOptions": {
    /* ... */
    "jsxFragmentFactory": "MyFragment"
  }
}
```

このように設定すると、`<>`記法は`<MyFragment>...</MyFragment>`と同じ意味になります。そのため、ランタイムもそれに合わせて変更する必要があります。