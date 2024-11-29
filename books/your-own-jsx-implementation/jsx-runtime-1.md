---
title: "独自のJSXランタイムをセットアップする"
---

前章のトランスパイル結果を再掲します。

```js:dist/index.js
import { jsx as _jsx } from "react/jsx-runtime";
const children = [
  _jsx("h1", { id: "hello", children: "Hello, world!" }),
  _jsx("p", { children: "Goodbye, world!" }),
];
const element = _jsx("div", { children: children });
```

上のトランスパイル結果を見ると、`jsx`や`jsxs`が`react/jsx-runtime`からインポートされています。このインポート元を書き換えることで、独自のJSXランタイムを使うことができ、JSXの結果を自由に操作できるようになります。

ということで、この章では独自のJSXランタイムを用意して、最低限動くようにしましょう。

## JSXランタイムのインポート元の指定

`jsx`といった関数のインポート元は、TypeScriptの設定で指定できます（他のトランスパイラを使っている場合はそちらの設定が必要になることもあります）。

今回は、同じプロジェクトの中にJSXランタイムがあるということにしましょう。この場合、[Subpath imports](https://nodejs.org/api/packages.html#subpath-imports)が使えます。

```json:tsconfig.json
{
  "compilerOptions": {
    /* ... */
    "jsxImportSource": "#my-jsx"
  }
}
```

こうすると、トランスパイル結果はこんな感じになります。

```js:dist/index.js
import { jsx as _jsx } from "#my-jsx/jsx-runtime";
const element = _jsx("h1", { children: "Hello, world!" });
```

ちゃんとpackage.jsonにsubpath importsを設定しておきましょう。

```json:package.json
{
  "name": "jsx-sample-project",
  "version": "1.0.0",
  "description": "",
  "type": "module",
  "main": "index.js",
  "imports": {
    "#my-jsx/jsx-runtime": "./dist/my-jsx/jsx-runtime.js"
  },
  "devDependencies": {
    "prettier": "^3.3.3",
    "typescript": "^5.7.2"
  }
}
```

これで後は、`src/my-jsx/jsx-runtime.ts`を作って`jsx`と`jsxs`を実装すれば動くはずです。

## JSXランタイムをとりあえず実装する

例えば、ただ単にJSXを文字列化するだけのランタイムを作ってみましょう。

```ts:src/my-jsx/jsx-runtime.ts
export function jsx(
  tag: string,
  { children, ...props }: Record<string, unknown>,
): string {
  const attributes = Object.entries(props)
    .map(([key, value]) => ` ${key}="${value}"`)
    .join("");
  const innerHTML = Array.isArray(children) ? children.join("") : children;
  return `<${tag}${attributes}>${innerHTML}</${tag}>`;
}

export { jsx as jsxs };
```

:::message
これはHTML文字列のようなものを生成する関数ですが、エスケープ処理などは省略しています。実際にはエスケープ処理を入れる必要があります。
:::

これで動くはずです。Node.jsで実行してみましょう。

```ts:src/index.tsx
const element = (
  <div>
    <h1 id="hello">Hello, world!</h1>
    <p>Goodbye, world!</p>
  </div>
);

console.log(element);
```

```sh
$ npx tsc
$ node dist/index.js
<div><h1 id="hello">Hello, world!</h1><p>Goodbye, world!</p></div>
```

この例では、`element`にすでに文字列が入っています。そのため、`console.log`でJSX式の結果を表示するとその文字列が表示されます。

これで独自のJSXランタイムを最低限実装することができました。