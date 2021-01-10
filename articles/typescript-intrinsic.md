---
title: "TypeScript 4.1で密かに追加されたintrinsicキーワードとstring mapped types"
emoji: "📂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

TypeScript 4.1では、Mapped typesにおけるkey remappingやtemplate literal typesに付随する新機能として、標準ライブラリに`Uppercase`などの型が追加されました。

```ts
// type Str = "FOOBAR"
type Str = Uppercase<"foobar">;
```

上の例から分かるように、`Uppercase`型は一つの文字列を受け取る型関数で、文字列のリテラル型を渡すとその文字列中の小文字を全て大文字にした文字列のリテラル型が返ります。他にも、`Lowercase`、`Capitalize`, `Uncapitalize`があります。

これらの型は標準ライブラリ（`lib/es5.d.ts`）にその定義があります。そこで使われているのが**intrinsicキーワード**なのです。以下はTypeScript 4.1時点の標準ライブラリからの引用です。

```ts
/**
 * Convert string literal type to uppercase
 */
type Uppercase<S extends string> = intrinsic;

/**
 * Convert string literal type to lowercase
 */
type Lowercase<S extends string> = intrinsic;

/**
 * Convert first character of string literal type to uppercase
 */
type Capitalize<S extends string> = intrinsic;

/**
 * Convert first character of string literal type to lowercase
 */
type Uncapitalize<S extends string> = intrinsic;
```

## intrinsicキーワードとは

これらの型定義を見ると、なんだか不思議な定義となっていますね。`=`の右が全て`intrinsic`となっています。これは**実装がコンパイラの内部実装として隠蔽されている**ことを意味しています。言い換えれば、TypeScriptの通常の型定義では表現できないような特殊な機能を表現するものです。JavaScriptをお使いの方は、組み込み関数の中身を表示しようとすると`function log() { [native code] }`のように表示されて関数の中身が隠されていたという経験がおありでしょう。それと似たようなものだと考えましょう。

### intrinsicキーワードの必要性

TypeScript 4.1では文字列の大文字・小文字変換の機能を実装することになりましたが、実は当初の実装では`intrinsic`を使っていませんでした。代わりに、次のように`uppercase`・`lowercase`・`capitalize`・`uncapitalize`という4つのキーワードによる新たな構文を定義され、それらを用いる方式となっていました（[#40336](https://github.com/microsoft/TypeScript/pull/40336)）。

```ts
// 古い方式
// type Str = "FOOBAR"
type Str = uppercase "foobar";
```

この古い方法の欠点は、新しい組み込み型関数を実装するたびに構文が増えてしまうことにありました。古い方法では、4種類の文字列操作を実現するためだけに4種類の新しい構文が追加されています。TypeScript（あるいはTypeScriptの新機能の実装を主に担うAnders Hejlsberg氏）はミニマルな機能で様々なユースケースに対応できるような機能設計を好む傾向にあり、文字列処理のためだけに存在する4種類の新しい構文というのはそれに逆行してしまうものでした。

代替となる`intrinsic`キーワードでは、新しい構文は`intrinsic`キーワードという1つだけでミニマルです。また、将来的な機能拡張もこの`intrinsic`キーワードで対応できます。

### intrinsicキーワードの挙動

TypeScript 4.1からは`intrinsic`が新たにキーワードとなりました。よって、TypeScript 4.1では次のようなコードはコンパイルエラーとなります（これはTypeScript 4.0では動作しました）。ちなみに、`type intrinsic = ...`はエラーにならずに単に無視されるようです。

```ts
type intrinsic = string;
// エラー: The 'intrinsic' keyword can only be used to declare compiler provided intrinsic types.
type A = intrinsic;
```

`intrinsic`はコンパイラ内部で定義される特殊な型ですが、その実態は`intrinsic`に与えられた**型名**と**型引数の数**によって決められます。例えば、次のようにすると自分で`intrinsic`を使うことができます。

```ts
function foo() {
    type Uppercase<T> = intrinsic;
    // type T = "FOO"
    type T = Uppercase<"foo">
}
```

わざわざ関数`foo`の中に入れているのはグローバルに定義されている`Uppercase`と重複しないように別のスコープとするためです。このように、`Uppercase`という名前で型引数を1つ持つ型エイリアス（`type`宣言）を`intrinsic`と宣言することにより、コンパイルエラーを起こさずに`intrinsic`を使うことができます。この処理はTypeScriptコンパイラの中で次のように書かれています（TypeScriptのコミット`eca8957430bf52cfdfa2bfe03c9431682da6a558`の`src/compiler/checker.ts`から引用。以降の引用も同じ）。

```ts
const type = getDeclaredTypeOfSymbol(symbol);
if (type === intrinsicMarkerType && intrinsicTypeKinds.has(symbol.escapedName as string) && typeArguments && typeArguments.length === 1) {
    return getStringMappingType(symbol, typeArguments[0]);
}
```

ここに登場した`intrinsicMarkerType`というのが`intrinsic`キーワードに与えられるコンパイラ内部の特殊な型で、`intrinsicTypeKinds`というのが`intrinsic`がサポートする名前の一覧です。現在は`Uppercase`や、そのほかに`Lowercase`、`Capitalize`、`Uncapitalize`がこれに含まれています。さらに`typeArguments`の数のチェックも行われており、型変数がちょうど1個でなければならないことが分かります。これはつまり、`type Uppercase<T> = intrinsic;`はOKだが`type Uppercase<U, T> = intrinsic;`のようなものは認識されないことを意味しています。これは現在`intrinsic`で定義される型関数が全て1引数だからこうなっているのであり、将来2引数などのintrinsic型関数が登場するような場合には適宜変更されると考えられます。

`Uppercase`型のコンパイラ内部での実体は、型定義に書かれている通り常に`intrinsicMarkerType`であり、`Uppercase<"foo">`のように`Uppercase`が使用された際に上に引用したロジックが動作し、`intrinsicMarkerType`が解決されます。

このコードに`getStringMappedType`とありますが、この関数の役割は主に2つあります。一つは、文字列リテラル型に即座に変換を適用することです。先ほどのコードで`T`が`"FOO"`型となるのはこちらの機能によるものです。

```ts
// type T = "FOO"
type T = Uppercase<"foo">
```

もう一つは、**string mapped type**という新たな型を生成するものです。こちらは、`Uppercase<T>`で`T`が別の型変数だったりした場合に発生します。String mapped typeというのは、conditional typeやindex access type、あるいはmapped typeなどと同レベルの概念だと思ってもらって構いません。ただし、ここで挙げたものはそれぞれ専用の構文を持つのに対し、string mapped typeは`intrinsic`を介してのみ発生します。

## String Mapped Type

ここで出てきたstring mapped typeについてさらに見ていきましょう。この型が存在する理由の一つは、`Uppercase`などの機能を型変数に対して使用できるようにすることです。

```ts
function toUpperCase<T extends string>(str: T) {
    type Mapped = Uppercase<T>;
    return str.toUpperCase() as Mapped;
}

// const str: "PIKACHU"
const str = toUpperCase("pikachu");
```

このコードでは関数`toUpperCase`の返り値は`Uppercase<T>`です。実際に`toUpperCase("pikachu")`としてこの関数を呼び出すと型引数`T`が`"pikachu"`型に推論され、その結果返り値の`Uppercase<T>`が`Uppercase<"pikachu">`になった結果として`"PIKACHU"`型に解決されます。このように、型引数を含む関数インターフェースの中でも`Uppercase`などの機能を使えるようにするために、string mapped typeが内部的に使われています。

### String mapped typeと型推論

TypeScriptの型の多くは**型推論**、特に**型引数の推論**をサポートしています。その典型例はtemplate literal typesです。

```ts
// type Rest = "def"
type Rest = "abcdef" extends `abc${infer S}` ? S : never;

// type Rest2 = never
type Rest2 = "aaaaaa" extends `abc${infer S}` ? S : never;
```

このように、TypeScriptは`` "abcdef" extends `abc${infer S}` ``のような条件を判定し、必要に応じて`S`を推論できる必要があります。ここでは`infer`を用いましたが、関数に与えられた引数から型引数を推論する場合も同様です。

ここで、`"abcdef"`や`"aaaaaa"`に対して`` `abc${infer S}` ``をマッチングして条件判定や`S`の推論を行うのはtemplate literal typeの機能の一部です。

では、string mapped typeの場合はどうでしょうか。実は、この場合型推論は全然行われません。

```ts
// type A = string
type A = "ABC" extends Uppercase<infer S> ? S : never;
// type B = never
type B = 123 extends Uppercase<infer S> ? S : never;
```

この結果は次のように理解できます。まず、`"ABC"`と`Uppercase<infer S>`をマッチングさせようとしても何の推論も起こりません。そのため、`S`については何の情報も得られないことになります。ただし、`Uppercase`が標準ライブラリ内で`type Uppercase<S extends string> = intrinsic;`と定義されていることにより、`S`は`extends string`という制約があることが分かります。`S`にはこれ以上の情報がないため`S`は`string`に置き換えられます。`A`の場合、`"ABC" extends Uppercase<string>`という条件判定が行われることになり[^note_1]、`Uppercase<string>`は`string`なのでこれは`"ABC" extends string`と同じであるため真となり、結果として`A`には`S`から置き換えられた`string`が入ります。一方の`B`は、`123 extends Uppercase<string>`が偽なので`never`となります。

[^note_1]: 厳密に言えば、`extends`の右は「`string`に対するstring mapped type」であり、「`Uppercase`という型エイリアスに`string`型引数を適用したもの」ではないのですが、前者を表す構文的な記法が無いのでここでは分かりやすさのために両者を同一視しています。

このことを裏付けるためには次のような実験をしてみましょう。適当な関数スコープの中で`Uppercase`を再定義しました。

```ts
function f() {
    type Uppercase<S> = intrinsic;

    // type A = unknown
    type A = "ABC" extends Uppercase<infer S> ? S : never;
    // type B = unknown
    type B = 123 extends Uppercase<infer S> ? S : never;
}
```

ここでの`Uppercase`の定義は標準ライブラリと異なり、`S extends string`という制約がありません。その結果、`Uppercase<infer S>`においても`S`に対する制約が一切なくなり、その結果`S`は`unknown`型と推論されています。

このように、型が`intrinsic`により定義されていても、標準ライブラリに露出している部分も`Uppercase`などの挙動に影響しています。なかなか面白いですね[^note_2]。

[^note_2]: この理由は`Uppercase<infer S>`における`S`のconstraintの決定が構文的に行われている（`infer S`が`Uppercase`の型引数であることを構文的に見て`Uppercase`の型引数のconstraintを見に行っている）からです。昔はこれが`Uppercase<(infer S)>`のようにカッコで囲むとうまくいかなくなるバグがあったのですが、[筆者が修正しました](https://github.com/microsoft/TypeScript/pull/40406)。

## まとめ

この記事では、TypeScript 4.1の新機能である`Uppercase`などの型を表現するのに使われている`intrinsic`キーワードの機構を解説しました。

`Uppercase<T>`の結果は内部的にstring mapped typesになりますが、それらにいちいち（`uppercase "foo"`のような）専用構文を与えるのは将来性の面で望ましくないため、`intrinsic`キーワードの機構を用いることでstring mapped typesの存在をコンパイラ内部に隠蔽したと見ることができます。その結果として、string mapped typesはそれを直接表現する構文を持たない（`Uppercase<T>`のように間接的に表現するしかない）特殊な型となりました。このような型は今度も増えていくのかもしれません。