---
title: "ts-array-lengthを支えるテクニック"
emoji: "🪜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

皆さんこんにちは。筆者は先日、TypeScript向けライブラリの**ts-array-length**を公開しました。

https://github.com/uhyo/ts-array-length

この記事ではこのライブラリを宣伝するとともに、ライブラリの実装がどのようになっているのか解説します。

# ts-array-lengthの機能

ts-array-lengthは3つの関数を提供しており、これらを使うことでなんと**配列の要素数をチェックできます**。

例えば`hasLength`を使うと、配列の要素がちょうど2個かどうか調べることができます。

```ts
if (hasLength(arr, 2)) {
  // arrは2要素の配列！
  const [first, last] = arr;
}
```

他にも、要素数がある数値以上かどうか判定する`hasMinLength`と、要素が1つでもあるかどうか判定する`isNonEmpty`があります。

```ts
if (hasMinLength(arr, 3)) {
  // arrは3要素以上
}

if (isNonEmpty(arr)) {
  // arrは1要素以上
}
```

めちゃくちゃ画期的ですね。ts-array-lengthが火星に到達する日も遠くないでしょう。

## ts-array-lengthがなぜ必要なのか

……というのは冗談ですが、こんな関数たちはなぜ必要なのでしょうか。それはもちろん、型チェックのためです。

とくに、これらの関数はTypeScriptのnoUncheckedIndexedAccessオプションが有効な場合に役に立ちます。このオプションが有効な状況では、配列に対してインデックスアクセスすると常に`undefined`の可能性があるものとして扱われます。

```ts
// arr: string[]

const first = arr[0];
//    ^? const first: string | undefined
```

そして、現在のところTypeScriptは`arr.length`に対してif文などでチェックしても、その結果を考慮してくれません。

```ts
if (arr.length > 0) {
  // まだ string | undefined になってしまう……
  const first = arr[0];
}
```

ということで、チェックすれば型が絞り込まれるように作ったのがts-array-lengthです。例えば`hasLength`の場合、次のような絞り込みが提供されます。

```ts
// ここではarrは string[] 型
if (hasLength(arr, 2)) {
  // ここではarrは readonly [string, string] 型
}
```

TypeScriptでユーザー定義の絞り込みを行うためには関数呼び出しを介する必要があるので、このように関数の形で提供しています。

絞り込まれたif文の中でarrに破壊的変更をしたら壊れてしまうとかそういう問題は無くはないのですが、それは大抵の絞り込みで言えることなので今回は目を瞑ります。🙈

## ts-array-lengthはいつ役立つのか

前述のように、ts-array-lengthはTypeScriptのnoUncheckedIndexedAccessオプションが有効な場合に役に立ちます。このオプションはstrictをtrueにしてもデフォルトではオンにならないという厳しいオプションですが、これをオンにしないと型に現れない`undefined`が出現するため、ぜひオンにすべきです。

常に`undefined`の可能性が付きまとうのは不便に思いますが、まずはそもそもインデックスアクセスを使うのをなるべく控えるべきです。代わりに、for-of文やforEach、mapといった方法を使えばundefinedは出てきません。しかし、それでもインデックスアクセスが必要になってしまう場面は存在します。

また、もともとnoUncheckedIndexedAccessが有効ではなかったコードベースに後からnoUncheckedIndexedAccessを有効にする場合は、インデックスアクセスに依存したコードが多くなってしまっていることがあります。その場合に最低限の修正で型エラーを消すためにもts-array-lengthが有効です。

# ts-array-lengthの裏側

ここからは、ts-array-lengthの実装がどのようになっているのか説明します。前述のように、型の絞り込みのためには関数が必要であり、関数の返り値を `引数名 is 型`という形（型述語）にすることで、その関数が`true`を返したら`引数名`が`型`に絞り込まれるという意味になります。

具体的には、`hasLength`, `hasMinLength`, `isNonEmpty` は次のように実装されています。

```ts
export function hasLength<T, N extends number>(
  arr: readonly T[],
  length: N,
): arr is ReadonlyArrayExactLength<T, N> {
  return arr.length === length;
}

export function hasMinLength<T, N extends number>(
  arr: readonly T[],
  length: N,
): arr is ReadonlyArrayMinLength<T, N> {
  return arr.length >= length;
}

export function isNonEmpty<T, N extends number>(
  arr: readonly T[],
): arr is ReadonlyArrayMinLength<T, 1> {
  return arr.length >= 1;
}
```

絞り込み先の型として使われている`ReadonlyArrayExactLength`と`ReadonlyArrayMinLength`がこのライブラリの本体です。`ReadonlyArrayExactLength<T, N>`は、要素数がちょうど`N`個ある`T`型の配列です。`ReadonlyArrayMinLength<T, N>`は、要素数が`N`個以上ある`T`型の配列です。このような型はTypeScriptのタプル型を用いることで実装できます。

## 指定した要素数のタプル型を作る

このような型を実装するための基本的なアイデアは次の記事で説明されていますが、改めて簡単に説明します。

https://qiita.com/uhyo/items/80ce7c00f413c1d1be56

例えば、要素数がちょうど2個のタプル型は `[T, T]`型、2個以上のタプル型は`[T, T, ...T[]]`と表現できます。

そこで、`ReadonlyArrayExactLength<T, 2>`のように、要素数を数値のリテラル型で与えれば上記の型が生成されるような型を実装することを考えます。

実装は、TypeScriptでは「タプル型に関して要素を1個追加した新しいタプル型を作れる」ということと、「タプル型の要素数はlengthプロパティの型を見ればリテラル型で取得できる」ことを利用して、型を再帰することで行えます。基本的な実装は次の通りになります。

```ts
export type ReadonlyArrayExactLength<T, N extends number> =
   ReadonlyArrayExactLengthRec<T, N, readonly []>;

type ReadonlyArrayExactLengthRec<
  T,
  L extends number,
  Result extends readonly T[],
> = Result["length"] extends L
  ? Result
  : ReadonlyArrayExactLengthRec<T, L, readonly [T, ...Result]>;
```

`ReadonlyArrayExactLengthRec`は、`Result`に与えられたタプル型が`L`個の要素を持っているかどうか判定し、違う場合は`Result`に1個付け足して再帰します。`Result`の要素が`L`個になったら再帰を終了して`Result`を返します。

計算はおおよそ次のように行われます。

```ts
ReadonlyArrayExactLength<T, 2>
= ReadonlyArrayExactLengthRec<T, 2, readonly []>
= ReadonlyArrayExactLengthRec<T, 2, readonly [T]>
= ReadonlyArrayExactLengthRec<T, 2, readonly [T, T]>
= readonly [T, T]
```

詳細は省略しますが、`ReadonlyArrayMinLength`のほうも同様の実装となっています。

## いろいろな使い方に耐えられるように改良する

以上が基本的なアイデアですが、ts-array-lengthに実装されているものはもう少し複雑な定義となっています。その理由は、変な使い方をされても大丈夫なようにです。

### ユニオン型の対応

まず、`N`がユニオン型だった場合を考えましょう。

```ts
ReadonlyArrayExactLength<string, 1 | 2 | 3>
```

この場合、「1要素か2要素か3要素」という意味なので`[string] | [string, string] | [string, string, string]`となるのを期待したいところですが、上の実装だとそうなりません。実際には`[string]`となってしまいます。その理由は、`Result`のlengthが`N`に到達したか判断するところで`N`が`1 | 2 | 3`なので、「lengthが1または2または3なら再帰を終了する」という意味になってしまうからです。

```ts
ReadonlyArrayExactLength<T, 1 | 2 | 3>
= ReadonlyArrayExactLengthRec<T, 1 | 2 | 3, readonly []>
= ReadonlyArrayExactLengthRec<T, 1 | 2 | 3, readonly [T]>
= readonly [T]
```

この挙動を改善するために有効なのがunion distributionです。これはユニオン型の各要素に対して別々に計算して、その結果をユニオン型にまとめることができる機構です。具体的には、次のように実装を修正します（`ReadonlyArrayExactLengthRec`の実装は同じなので省略）。

```ts
export type ReadonlyArrayExactLength<T, N extends number> =
  N extends number
  ? ReadonlyArrayExactLengthRec<T, N, readonly []>
  : never;
```

実装としては、`N extends number`という条件分岐（条件型; conditional type）が入りました。実は、そもそも型引数の制約に`N extends number`と書いてあるので、条件分岐としては常にtrueになり意味がありません。

TypeScriptでは、このようにconditional typeの条件部分の形が「`型引数 extends 型`」である場合はunion distributionが起こります。その場合、`型引数`がユニオン型であれば、ユニオン型のそれぞれの構成要素に対して別々に結果が計算され、それらがユニオン型でまとめられるのです。今回の場合、`N`が`1 | 2 | 3`であれば、`N`が`1`の場合と`2`の場合と`3`の場合にばらして型の計算が行われることになります。

つまり、新しい実装では`ReadonlyArrayExactLength<T, 1 | 2 | 3>`は次のように計算されます。

```ts
ReadonlyArrayExactLength<T, 1 | 2 | 3>
= ReadonlyArrayExactLengthRec<T, 1, readonly []> | ReadonlyArrayExactLengthRec<T, 2, readonly []> | ReadonlyArrayExactLengthRec<T, 3, readonly []>
```

これで意図した挙動にすることができました。このように、TypeScriptではunion distributionを使うことでユニオン型をかなり柔軟に取り扱うことができ、この記事のような複雑な型を実装する場合に役立ちます。

### `N`に`number`が入ってきたときの対応

今回実装した型は、`N`に具体的な数が入ってきた場合に意味を成します。そのため、`ReadonlyArrayExactLength<T, number>`のような指定にはあまり意味がありません。しかし、汎用性の高い型とするためにはこのような場面にも対応しなければいけません。

`ReadonlyArrayExactLength<T, number>`という型は、lengthが`number`なら良いという意味なので、実質何も制限をかけていないと解釈するのが妥当そうです。つまり、この場合は`readonly T[]`を結果とすれば良さそうですね。このケースのための分岐を入れる必要があるので、次のようにします。

```ts
export type ReadonlyArrayExactLength<T, N extends number> =
  number extends N
  ? readonly T[]
  : N extends number
  ? ReadonlyArrayExactLengthRec<T, N, readonly []>
  : never;
```

追加されたのは最初の「`number extends N`」という分岐です。これは「`number`が`N`の部分型である」という意味なので、`N`は`number`全体を受け入れられる型でないといけません。今回の場合は`N extends number`という制約もかかっているので、これで`N`が`number`であるという条件になっています。

### `N`が非負整数ではないときの対応

これまで、`N`には非負整数が入るという前提で話していましたが、実際には`-3`とか`0.5`といった値が`N`に与えられる可能性もあります。この場合に対処しなければいけません。

今の実装では、次のようなエラーが出ます。これは上限までループしても型の計算が終わらなかったという意味です。今回の場合、型の計算が無限ループに入ってしまうためこのエラーが出ます。

```ts
// エラー: Type instantiation is excessively deep and possibly infinite.(2589)
type A = ReadonlyArrayExactLength<string, -3>
```

`ReadonlyArrayExactLengthRec`の計算では最初に`Result`のlengthが0から始まり、1、2、3、……と進んで`N`に一致するまで`Result`の要素数を増やし続けます。`N`が`-3`や`0.5`だった場合は`Result`のlengthがずっと一致しないため無限ループになってしまいます。

配列のlengthが`-3`や`0.5`などになることはないので、非負整数以外の数値型が`N`として与えられたときは`ReadonlyArrayExactLengthRec`の計算をしないようにする必要があります。そのような値が存在しないという意味で`never`にしてもよいですが、何となく怖いのでこちらも余計な制限をかけない`readonly T[]`を結果にしておきましょう。

そうなると、`N`という数値のリテラル型が、非負整数かどうかを判定する必要があります。これを`IsCertainlyInteger<N>`として実装してみましょう。ただ、TypeScriptは型レベルの数値計算をサポートしていないので、これは一筋縄ではいかず、完璧な判定はできません。しかし、テンプレートリテラル型を使えばある程度の判定は可能です。それが次の実装です。

```ts
export type IsCertainlyInteger<N extends number> =
  IsCertainlyIntegerImpl<`${N}`>;

type IsCertainlyIntegerImpl<Str extends string> =
  Str extends `${infer _}.${infer _}`
    ? false
    : Str extends `-${infer _}`
    ? false
    : Str extends "Infinity" | "-Infinity" | "NaN"
    ? false
    : true;
```

この実装では、`IsCertainlyInteger`は、与えられた`N`を文字列のリテラル型`` `${N}` ``に変換して`IsCertainlyIntegerImpl`に渡しています。TypeScriptは文字列に対する型レベル計算はできるので、文字列にして判定しようという魂胆です。例えば`N`が`3`というリテラル型だった場合、文字列型への変換後は`"3"`という型になります。同様に、`-3`であれば`"-3"`に、`0.5`であれば`"0.5"`になります。

それを踏まえて上の実装を見ると、「文字列に変換した後、`.`を含んでいる文字列や`-`で始まる文字列、`Infinitely`や`NaN`は弾く」というものになっています。これで負の数や小数は弾けます。ただ、`1.23e+100`のような大きな値を間違えて弾いてしまうことがあります。今回は、そのような大きな数値のユースケースがないので問題ありません。

上の型関数を組み込んだ`ReadonlyArrayExactLength`は次のようになります。これで、負の数や小数が`N`として与えられても無限ループにはなりません。

```ts
export type ReadonlyArrayExactLength<T, N extends number> =
  number extends N
  ? readonly T[]
  : N extends number
  ? IsCertainlyInteger<N> extends true
    ? ReadonlyArrayExactLengthRec<T, N, readonly []>
    : readonly T[]
  : never;
```

ライブラリとして便利な型関数を提供する場合、このようになるべく色々な入力に耐えられるようにするとよいでしょう。

# 実装の欠点

実際のts-array-lengthの実装は上記の工夫を全部盛り込んでいますが、一応欠点はあります。例えば、`ReadonlyArrayExactLength<string, 1000>`のように大きな整数を`N`として与えると再帰の上限に達してエラーとなってしまいます。タプル型の生成の効率を上げれば上限を増やせそうですが、そこまで大きな数のサポートはあまり必要なさそうなので今回は省いています。

# まとめ

今回は、筆者がとある事情で作ったts-array-lengthを紹介し、その実装について詳しく説明しました。このような型レベルの実装は体系的な説明がしにくいのですが、具体例を通じてみなさんの参考になれば幸いです。

https://github.com/uhyo/ts-array-length