---
title: "JSXに型を付ける"
---

前章までで実装してきたコードは、トランスパイルすれば動きはするものの型チェックが通りませんでした。例えばこんなエラーが発生します。

```
src/index.tsx:2:3 - error TS7026: JSX element implicitly has type 'any' because no interface 'JSX.IntrinsicElements' exists.

2   <div>
    ~~~~~
```

これは、JSXに対する型定義がされていないことを意味するエラーです（`implicitly has type 'any'`となっているので、`noImplicitAny`を前提とするエラーになっています）。

ということで、この章ではTypeScriptの機能を使ってJSXに型を付けていきましょう。基本的に、以下のページに書いてあることに従えば大丈夫です。

https://www.typescriptlang.org/docs/handbook/jsx.html

## 組み込み要素を定義する

JSXの構文上、大文字で始まるタグ名と小文字で始まるタグ名では意味が変わってきます。大文字で始まるタグ名はプログラム上で定義されたコンポーネントを意味しており、小文字で始まるタグ名は組み込み要素を意味します。これまでに出てきた例は全て組み込み要素でした。

トランスパイル結果を見ると分かるように、組み込み要素はトランスパイル後は`"div"`のような文字列として表され、`jsx`の第一引数に与えられます。

TypeScriptでは、組み込み要素に対する型定義が可能です。具体的には、`JSX.IntrinsicElements`というインターフェースを使います（これは`JSX`という名前空間に属する`IntrinsicElements`型という意味です）。このインターフェースは、組み込み要素の名前をキーとして、その要素に渡せるpropsの型を定義します。

ここでは、`div`と`h1`と`p`だけを定義してみましょう。

```tsx:src/my-jsx/types.d.ts
interface HasChildren {
  children?: string | string[];
}

declare namespace JSX {
  interface IntrinsicElements {
    div: HasChildren;
    h1: HasChildren;
    p: HasChildren;
  }
}
```

:::message
この`types.d.ts`がimportまたはexport宣言を含まないことが重要です。その場合`JSX`がグローバルな名前空間として定義されます。こうしないとJSXの型定義としては有効になりません。

もしimportまたはexportが必要な場合は`declare global`構文を使うことでグローバルな名前空間を定義できます。
:::

これで、TypeScriptに「組み込み要素は`div`と`h1`と`p`だけだよ」ということが伝わりました。今のところ、propsの型は`HasChildren`にしています。これは、`children`があってもいいし無くてもいい、ある場合は`string`か`string[]`であるということを意味します。前章で見たように、要素の子として複数のものが与えられた場合は配列になりますので、このような定義が必要です。

こうすると多くの型エラーは解消できますが、1つエラーが残っています。

```
src/index.tsx:3:9 - error TS2322: Type '{ children: string; id: string; }' is not assignable to type 'HasChildren'.
  Property 'id' does not exist on type 'HasChildren'.

3     <h1 id="hello">Hello, world!</h1>
          ~~
```

これは、h1という組み込み要素には`id`というpropsは無いよという型エラーです（正確には、型が間違っているというよりは、excess property checkに由来するエラーです）。

このエラーを解消するためには、`h1`のpropsに`id`を追加します。

```tsx:src/my-jsx/types.d.ts
declare namespace JSX {
  interface IntrinsicElements {
    div: HasChildren;
    h1: HasChildren & { id?: string };
    p: HasChildren;
  }
}
```

こうすれば、`<h1 id="hello">`に対して型エラーが出なくなります。これで、型チェックにも対応したJSXの実装を作ることができました。

## JSX式の結果の型

ところで、このコードで変数`element`の型はどうなっているでしょうか。

```tsx
const element = (
  <div>
    <h1 id="hello">Hello, world!</h1>
    <p>Goodbye, world!</p>
  </div>
);
```

ランタイムには`element`は文字列なので、`element`の型は`string`となるのが望ましいです。

しかし、実際に調べてみると、`any`になってしまいます。これは、JSX式の結果が何であるかということを定義していないからです。

これを定義する方法は簡単です。`JSX`名前空間の中に`Element`という型を定義すればいいです。現時点の全体を示します。

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
}
```

こうすることで変数`element`の型は`string`になりました。完璧ですね。

このように、JSX式の結果の型は`JSX.Element`で定義できます。

## ポイント

前章では`src/my-jsx/jsx-runtime.ts`としてJSXのランタイム実装を作りましたが、この章で用意したJSX用の型定義はそれとは独立していました。

このように、TypeScriptではJSXのランタイムと型定義は切り離されています。