---
title: "コンポーネントを拡張する"
---

今までのところ、コンポーネントは「propsを受け取って`MyJSXElement`を返す関数」でした。実はTypeScriptは他にもクラスコンポーネントをサポートしているのですが、今回は省略して引き続き関数コンポーネントのみを取り扱います。

## コンポーネントの呼び出しに対する型チェック

TypeScriptが関数コンポーネントの呼び出し（つまり`<Title>...</Title>`のようなJSX式）に行ってくれるチェックを正確に述べると、以下のことをチェックしてくれます。

- そのJSX式の属性たちを集めたオブジェクト（props）が、コンポーネント（`Title`）の引数の型に適合していること
- コンポーネントの返り値の型が`JSX.Element`になっていること

前章の型定義を使って、これらに違反すると型エラーになることを確認してみましょう。まずは引数の型です。

```tsx:src/index.tsx
const Title = (props: { text: string }) => <h1>{props.text}</h1>;

const element = <Title text={123} />;
```

こうすると、次のようなエラーが出ます。

```
src/index.tsx:3:24 - error TS2322: Type 'number' is not assignable to type 'string'.

3 const element = <Title text={123} />;
                         ~~~~

  src/index.tsx:1:25
    1 const Title = (props: { text: string }) => <h1>{props.text}</h1>;
                              ~~~~
    The expected type comes from property 'text' which is declared here on type '{ text: string; }'
```

`Title`の`text` propsは`string`型を受け取るように定義されているので、`number`型を渡すと型エラーになります。これは普通の関数呼び出しのチェックと同じなので分かりやすいですね。

次に返り値の型です。

```tsx:src/index.tsx
const Title = (props: { text: string }) => `## ${props.text}`;

const element = <Title text="Hello, world!" />;
```

こうすると、次のようなエラーが出ます。

```
src/index.tsx:3:18 - error TS2786: 'Title' cannot be used as a JSX component.
  Its return type 'string' is not a valid JSX element.

3 const element = <Title text="Hello, world!" />;
```

エラーメッセージは、`Title`はJSXコンポーネントとして使えないよという内容になりました。TypeScriptでは、JSXコンポーネントというのは返り値の型が`JSX.Element`となっている（正確に言えば`JSX.Element`に代入可能になっている）関数のことを指します。この`Title`関数は`string`を返しているので、JSXコンポーネントとして使えません。

### nullの取り扱い

ただし、注意すべきことがあります。それは、実は`null`を返す関数もJSXコンポーネントとして認められるということです。

```tsx:src/index.tsx
// 型エラーにならない！
const Title = (props: { text: string }) => null;

const element = <Title text="Hello, world!" />;
```

つまり、先ほど説明した返り値のチェックは、より厳密に言えば次のようになります。

- コンポーネントの返り値が`JSX.Element`か`null`であること

これは、Reactにおいてコンポーネントが何も表示しない場合に`null`を返す慣習となっていることに合わせたものでしょう。

## 関数コンポーネントの返り値を拡張する

ここからがこの章の本題です。実は、比較的最近（TypeScript 5.1）になって`JSX.ElementType`という新機能が追加されました。これは、関数コンポーネントの型をより自由に指定できるようにするものです。

例えば、今回実装したJSXランタイムでは、関数コンポーネントは`Renderable`を返せば動く実装になっています。つまり、関数コンポーネントが文字列とか`undefined`とかを返しても動くはずだということです。しかし、上述のように型チェックで弾かれていました。

ということで、`JSX.ElementType`を指定して、関数コンポーネントは`Renderable`を返せばいいという定義にしてみましょう。

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
    // 追加
    type ElementType = string | ((props: any) => Renderable);

    interface ElementChildrenAttribute {
      children: unknown;
    }
  }
}
```

このように、`JSX.ElementType`には、JSXのタグとして使用できる値そのものの型を指定します。`<h1>`のような組み込み要素をサポートするために`string`を許容し、関数コンポーネントは`(props: any) => Renderable`で表現しています。

この定義にすることで、先ほどは弾かれていたこのようなコードも型チェックが通るようになります。

```tsx:src/index.tsx
import { render } from "#my-jsx/jsx-runtime";

const Title = (props: { text: string }) => `## ${props.text}`;

const element = <Title text="Hello, world!" />;

console.log(render(element));
```

ただし、`Title`の関数としての返り値が`string`だとしても、JSX式の結果である`element`の型は依然として`JSX.Element`であることに注意してください。

言い換えれば、`JSX.Element`が従来「JSX式の結果」と「関数コンポーネントの返り値」の両方を表していたところ、`JSX.ElementType`の導入によって後者がこちらに分離されたということです。

`JSX.ElementType`の引数の型が`any`なのがちょっと心配になりますが、安全面では心配いりません。従来通り、propsの型が合わない場合は型エラーになります。

```tsx:src/index.tsx
const Title = (props: { text: string }) => `## ${props.text}`;

// 型エラーになる！
const element = <Title text={123} />;
```

つまり、`JSX.ElementType`はあくまで前段のチェックであり、propsの型のチェックはこれまでのように行われるということです。

### `JSX.ElementType`のReactでの活用

Reactもまた、`JSX.ElementType`の恩恵を受けています。Reactでは、関数コンポーネントが`undefined`や文字列、配列などを返してもランタイムでは動作しますが、従来はこのようなコンポーネントは型チェックで弾かれていました。型エラーが出るのでコンポーネントの返り値を無意味に`<>...</>`で囲むといった経験をされた方も多いのではないでしょうか。

`JSX.ElementType`の導入により、そのようなコンポーネント定義でも型チェックが通るようになりました。ちょっとしたわずらわしさが解消される嬉しいアップデートです。

また、Reactには非同期コンポーネントを追加したいという思惑もありました。非同期コンポーネントでは必然的に`Promise`を返すことができますが、この型チェックを通すためにも`JSX.ElementType`が必要です。TypeScriptに`JSX.ElementType`が追加されるに至った経緯としてはこれが大きく影響しているでしょう。

## 非同期コンポーネントのサポートを追加してみる

ということで、我々のランタイムにも非同期コンポーネントのサポートを追加してみましょう。何も見ずに自分で実装できたらJSXのことをかなり理解できています。

ルートとしては、`Renderable`に`Promise`を追加してしまうルートとしないルートが考えられますが、ここではしないルートを説明します。このルートでは、関数コンポーネントの返り値としてのみ`Promise`を受け付けます。

まずランタイムを修正します。

```ts:src/my-jsx/jsx-runtime.ts
export type FunctionComponentResult = Renderable | Promise<Renderable>;

export interface MyJSXElement {
  tag: string | ((props: Record<string, unknown>) => FunctionComponentResult);
  props: Record<string, unknown>;
}

export function jsx(
  tag: string | ((props: Record<string, unknown>) => FunctionComponentResult),
  props: Record<string, unknown>,
): MyJSXElement {
  return { tag, props };
}

export { jsx as jsxs };

export type Renderable =
  | string
  | MyJSXElement
  | Renderable[]
  | null
  | undefined;

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

  if (typeof tag === "function") {
    return render(await tag(props));
  }

  const { children, ...rest } = props;
  const attributes = Object.entries(rest)
    .map(([key, value]) => ` ${key}="${value}"`)
    .join("");
  const innerHTML = await render(children as Renderable);
  return `<${tag}${attributes}>${innerHTML}</${tag}>`;
}
```

一気に全部見せましたが、ポイントは`FunctionComponentResult`型の定義と`render`関数の修正です。`FunctionComponentResult`は関数コンポーネントが返す型を表しています。`Promise<Renderable>`という型を追加することで、関数コンポーネントが`Promise`を返すことを許容するようになります。

`render`関数は、関数コンポーネントが`Promise`を返す場合に備えて、関数コンポーネントの呼出結果には`await`を噛ませています。`render`関数がasync関数となりましたが、`Promise`を返す関数コンポーネントをサポートするためには必然的にこうなります。

次に型定義の修正です。やることは簡単で、先ほど定義した`FunctionComponentResult`を使って関数コンポーネントの型を修正するだけです。

```ts:src/my-jsx/types.d.ts
import type { MyJSXElement, Renderable, FunctionComponentResult } from "./jsx-runtime";

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
    type ElementType = string | ((props: any) => FunctionComponentResult);

    interface ElementChildrenAttribute {
      children: unknown;
    }
  }
}
```

これで非同期コンポーネントをサポートするランタイムができました。非同期コンポーネントを使ってみましょう。

```tsx:src/index.tsx
import { render } from "#my-jsx/jsx-runtime";

const Title = async (props: { text: string }) => {
  await new Promise((resolve) => setTimeout(resolve, 1000));
  return <h1>{props.text}</h1>;
};

const element = <Title text="Hello, world!" />;

console.log(await render(element));
```

この例では`Title`のレンダリングに1秒かかりますが、1秒後にちゃんと`<h1>Hello, world!</h1>`が表示されるはずです。

次のように他のコンポーネントの子として使っても問題ありません。

```tsx:src/index.tsx
import { render } from "#my-jsx/jsx-runtime";

const Title = async (props: { text: string }) => {
  await new Promise((resolve) => setTimeout(resolve, 1000));
  return <h1>{props.text}</h1>;
};

const element = (
  <div>
    <Title text="Hello, world!" />
    <p>Goodbye, world!</p>
  </div>
);

console.log(await render(element));
```

ということで、この章ではコンポーネントを拡張する練習として非同期コンポーネントのサポートを追加してみました。
