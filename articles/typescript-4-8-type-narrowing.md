---
title: "TypeScript 4.8で入る型の絞り込みの改善とは"
emoji: "🎢"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

皆さんこんにちは。今回はTypeScriptの更新先取りシリーズです。TypeScriptの次のバージョンでは、以下のPRの更新が入ると思われます。もちろんPRの著者はAndersさんです。このPRではTypeScriptの根幹を成す機能の一つである「型の絞り込み」が改善されます。特に、`unknown`型と`{}`型の取り扱いが修正されている点が注目に値します。

https://github.com/microsoft/TypeScript/pull/49119

# 型引数に対する推論が抱えていた既存の問題

`{}`型は、「`null`と`undefined`以外の任意の値」という意味を持つ型です。この型は形としては空のオブジェクト型ですが、JavaScriptでは`null`と`undefined`以外のプリミティブ（文字列や数値など）に対してもプロパティアクセスをしてもエラーにならないという仕様を考慮して、`{}`型には文字列や数値などのプリミティブも含まれています。

従来型引数に対する推論が抱えていた問題とは、任意の型引数が`{}`型に代入可能であることです。これにより、`null`や`undefined`が`{}`型に入ってしまう場合がありました。

```ts
function someFunc<T>(x: T) {
    // エラーにならない！
    const some: {} = x;
}

someFunc(null);
```

TypeScript 4.8ではこの挙動が修正され、上記のコードはエラーになります。エラーを修正するには、次のように`T`に制約をつけて`null`や`undefined`を防ぐ必要があります。

```ts
function someFunc<T extends {}>(x: T) {
    // OK
    const some: {} = x;
}
// こちらがエラーになるので安全！
someFunc(null);
```

この辺りの背景としては、任意の値を表す`undefined`型が`{}`型に比べて新参であるという事情があります。そのため、歴史的経緯から`extends`を持たない型引数は暗黙のうちに`extends {}`とみなされていたことになります。今回、それが修正されてあるべき姿になりました。ちなみに、TypeScript 4.7以前でも`T extends unknown`と明示的な制約をつければ`T`に`null`や`undefined`の可能性があると認識されます。

# `unknown`を`{}`に絞り込めるようになった

`unknown`型は`null`や`undefined`も含めて何でもあり得る値であり、まともに`unknown`型の値を使うには型の絞り込みを使う必要があります。例えば、`typeof x === "string"`とすれば`x`は`unknown`型から`string`型に絞り込まれます。

従来、`unknown`から`null`や`undefined`の可能性だけを除外することはできませんでした。

```ts
function someFunc(x: unknown) {
  if (x !== null && x !== undefined) {
    // TypeScript 4.7ではxがunknownのままのためエラー
    const y: {} = x;
  }
}
```

しかし、今回の修正により、この場合`x`が`{}`型に絞り込まれるようになります。

```ts
function someFunc(x: unknown) {
  if (x !== null && x !== undefined) {
    // TypeScript 4.8ではエラーにならない！
    const y: {} = x;
  }
}
```

ちなみに、`x !== null`だけだったりしてもうまく絞り込まれます。

```ts
function someFunc(x: unknown) {
  if (x !== null) {
    // TypeScript 4.8では {} | undefined 型になる
    x;
  }
}
```

以上の挙動は、型の絞り込みにおいて`unknown`が`{} | null | undefined`のように扱われるようになったと解釈できます[^note_unknown_def]。

[^note_unknown_def]: ただし、`unknown`が実際にユニオン型として再定義されたというわけではありません。そうしてしまうとunion distributionの挙動にも影響を与えてしまうので、恐らくできないのでしょう。

また、`!== null`や`!== undefined`以外の手段でも`unknown`を絞り込める可能性があります。例えば、`if (x)`のように真偽値チェックをする場合です。

```ts
function someFunc(x: unknown) {
  if (x) {
    // TypeScript 4.7ではunknown型のまま、
    // TypeScritp 4.8では{}型
    const y: {} = x;
  }
}
```

# 型引数に対する絞り込みの改善

型引数に対しても、`{}`にまつわる絞り込みの改善が行われています。

```ts
function someFunc<T extends unknown>(x: T) {
  if (x !== null && x !== undefined) {
    // TypeScript 4.7ではxはT型のまま
    // TypeScript 4.8ではxはT & {}型なのでOK
    const y: {} = x;
  }
}
```

# `NonNullable`の定義の改善

以上のような一連の変更により可能になったことがあります。それは組み込みの`NonNullable`型の定義の改善です。これが一連の変更によって成し遂げたかったことなのだと推測されます。`NonNullable<T>`という型は、その名前が示すように`T`から`null | undefined`の可能性を除いた型です。

従来の`NonNullable<T>`は次のような定義でした。

```ts
type NonNullable<T> = T extends null | undefined ? never : T;
```

これはconditional typesとunion distributionを駆使した型定義で、例えば`{ x: number } | null`のような型に対してはうまく動きます。

```ts
type ObjOrUndefined = { x: number } | undefined;

type Obj = NonNullable<ObjOrUndefined>; // { x: number }
```

しかし、この定義には2つの問題がありました。一つは、`unknown`型がユニオン型ではないためうまく動かないことです。

```ts
type A = NonNullable<unknown>; // TypeScript 4.7ではunknownのまま
```

もう一つは、型引数に対して`NonNullable`を使っても、型引数の中身がわからないためconditioanl typeが未解決のままになることです。

```ts
function someFunc<T>(x: T) {
  type A = NonNullable<T>; // TypeScript 4.7では T extends null | undefined ? never : T のまま
}
```

Conditional typeが未解決のままの場合、代入可能性などの判定が難しいため多くの操作が型エラーとなってしまいます。

```ts
function someFunc<T>(x: T) {
  type A = NonNullable<T>; // TypeScript 4.7では T extends null | undefined ? never : T のまま
  if (x !== null && x !== undefined) {
      // TypeScript 4.7では型エラーになってしまう
      const obj: A = x;
  }
}
```

TypeScript 4.8では、`NonNullable`の定義が大胆に変更されます。新しい定義はこうです。

```ts
type NonNullable<T> = T & {};
```

これにより、上記の2つの問題が解決されます。

```ts
type A = NonNullable<unknown>; // TypeScript 4.8では{}になる

function someFunc<T>(x: T) {
  type A = NonNullable<T>; // TypeScript 4.8ではT & {}
  if (x !== null && x !== undefined) {
    // TypeScript 4.8では型エラーにならない
    const obj: A = x;
  }
}
```

例の後半のコードから分かるように、この新しい定義が自然に動くためには`{}`に関する絞り込みの改善が必要でした。

新しい定義は型推論との親和性が高く、例えば次のようなコードが`as`などの補助無しにコンパイル可能になります。

```ts
function removeNullish<T>(value: T): NonNullable<T> {
  if (value === null || value === undefined) {
    throw new Error("Huh?")
  }
  return value;
}
```

# `{} | null | undefined`が何でも受け入れるようになった

関連する話題として、`{} | null | undefined`型に対する取り扱いが改善されます。この型は実質あらゆる値を受け入れるため`unknown`型と同様ですが、従来は`unknown`型をこの型に代入することはできませんでした。これは、前者がユニオン型であり後者がユニオン型ではないからです。TypeScriptでは`{} | null | undefined`に対する特別な処理が追加され、`unknown`をこの型に代入できるようになりました。

```ts
type PseudoUnknown = {} | null | undefined;

function someFunc(x: unknown) {
  // TypeScript 4.7ではエラー
  // TypeScript 4.8ではエラーにならない
  const y: PseudoUnknown = x;
}
```

このような変更によって、`unknown`の挙動が`{} | null | undefined`にさらに近くなり、`{}`の「`null`と`undefined`以外全部」という側面がさらに強調されることになります。

# その他の話題

ところで、次のような型はどのような挙動をとるでしょうか。

```ts
type A = string & {};
```

`{}`が`string`を完全に含んでいることを考えると、`A`は`string`になりそうです。しかし、実際は`A`は`string & {}`という型のままです（これはTypeScript 4.8でも変わりません）。

この変な挙動の理由は、プリミティブと`{}`のインターセクション型が次のようなハックに使われているからです（PRから引用）。

```ts
type Alignment = string & {} | "left" | "center" | "right";
```

この型は、意味的にはただの`string`と同様に任意の文字列を受け入れますが、`Alignment`型の引数や変数に対しては上の3種類の文字列が入力補完として現れるという挙動になります。このハックによって、任意の文字列を受け入れることと特定の文字列の補完が出ることを両立できるのです。`string & {}`を`string`にすると`| "left" ……`の部分が消えてただの`string`になってしまいます。このハックの挙動を保存するために`string & {}`を`string`にすることはできないのです。ハックではなく公式の方法を用意しようという議論もありますが、具体的な進展はないようです。

これを踏まえて、次のコードを見てみましょう。次のコードはTypeScript 4.8で挙動が変わります。

```ts
function nonNullable<T>(x: T): T & {} {
  if (x === null || x === undefined) {
    throw new Error("Huh?");
  }
  return x as T & {};
}

// TypeScript 4.7 では string & {} 型
// TypeScript 4.8 では string 型
const str = nonNullable<string>("");
```

このように、TypeScript 4.8では`T & {}`が`T`に`string`に入ることで`string & {}`になるような場合は、ただの`string`にされます。一方で、明示的に`string & {}`と書いた場合、前述のハックを維持するためにそのままになります。生まれてしまったハックは維持しつつもなるべく変な挙動を無くそうという配慮が感じられますね。

# まとめと感想

この記事では、TypeScript 4.8で入る型の絞り込みの改善について説明しました。結果的には、`NonNullable<T>`の定義の改善が重要です。そのために必要な一連の修正が行われたと理解するのがよいでしょう。

ただ、個人的には`unknown`が`{}`に絞り込まれても嬉しい場面がそれほどありません。

自分はよく次のような関数を作ります。これは`unknown`を（`{}`ではなく）`Record<string, unknown>`に絞り込みます。

```ts
function isNonNullish(value: unknown): value is Record<string, unknown> {
  return value !== null && value !== undefined;
}
```

こちらの方が、得られたオブジェクトに対して自由にプロパティアクセスができて便利です。ただ、TypeScriptではこのように「存在しないプロパティにアクセスできる」という挙動をデフォルトにする気は今のところは無いようです。それでもより快適なTypeScriptライフに一歩また近づきますね。