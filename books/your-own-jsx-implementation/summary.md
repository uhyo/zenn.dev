---
title: "まとめ"
---

この本では、自前のJSX実装を作るために必要な知識を紹介しました。

TypeScriptにおけるJSXの自前実装は、JSX式のトランスパイル先である`jsx`関数などの実装と、型チェックのための`JSX`名前空間の型定義の2つが必要です。両者を辻褄が合うように実装することで、TypeScriptに対応したJSXの実装を作ることができます。

この本ではプロジェクト内にJSXのランタイムを用意したのでインポート元が`#my-jsx/jsx-runtime`のようになっていましたが、独立したライブラリとして後悔したい場合もあるでしょう。その場合も、`your-library/jsx-runtime`のようなパスでJSXランタイムがインポートできるようにすればOKです。そして、`your-library`の型定義内に`namespace JSX`を含めましょう。

この本が、自前のJSX実装を作るための一助となれば幸いです。

## 完成品

```json:tsconfig.json
{
  "compilerOptions": {
    "target": "esnext",
    "jsx": "react-jsx",
    "jsxImportSource": "#my-jsx",
    "module": "esnext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "rootDir": "./src",
    "outDir": "./dist"
  },
  "include": ["src"]
}
```

```tsx:src/my-jsx/jsx-runtime.ts
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

export function Fragment({ children }: { children?: Renderable }): Renderable {
  return children;
}
```

```tsx:src/my-jsx/types.d.ts
import type {
  MyJSXElement,
  Renderable,
  FunctionComponentResult,
} from "./jsx-runtime";

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
    type ElementType = string | ((props: any) => FunctionComponentResult);

    interface ElementChildrenAttribute {
      children: unknown;
    }
  }
}
```

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

