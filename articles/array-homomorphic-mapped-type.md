---
title: "any、またお前か——配列とhomomorphic mapped typeの罠"
emoji: "🗺️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

TypeScriptは企業によって開発されてはいるもののなかなか大きなOSSの一つであり、openなissueの数はこの前5,000を超えました。日々いくつものissueが作られ、そして一部は閉じられていきます。TypeScriptはなかなか大きなOSSですから、issueが閉じられなかったとしても厳しい行く末を迎えるものは多くあります。TypeScriptチームが興味をそそられなかったならば、提案はSuggestionラベルとAwait More Feedbackラベルが与えられ、たとえ100を超える👍を得ようとも、奇跡でも起きなければ二度と掘り起こされることはありません（奇跡というのは、数年後にAndersさんが気まぐれにTypeScriptのおもしろい新機能を実装してそのついでに解決されるといったことを指します）。また、バグに関しても喫緊でないものはbugラベルをつけられてBacklogに放り込まれ、TypeScriptへのcontributionチャンスを伺うOSSコントリビューターの目に留まるまではその後何も起こらないでしょう。

色々なissueの中で幸運にもTypeScriptチームが興味を持ったものについては、TypeScriptチームの誰かがアサインされたり、具体的なマイルストーンが与えられたりします。また、ものによってはTypeScriptチーム内のDesign Meetingという場で方向性が議論されるようです。Design Meetingの議事録は不定期にGitHub上で公開され、なかなか面白い読み物です。

この記事では、そんなDesign Meetingで取り上げられた問題とその解決策が面白かったので紹介します。一言でいうと、結果が配列となることを意図したhomomorphic mapped typeだとしても、`any`が与えられてしまうと非配列型が与えられてしまうという問題です。

https://github.com/microsoft/TypeScript/issues/46247

# Homomorphic Mapped Typeとは

TypeScriptに結構詳しい人でないと、**homomorphic mapped type**というのはご存知ではないかもしれません。これは特定の形をしたmapped typeを指す用語であり、おおよそ次のような形のmapped typeがhomomorphic mapped typeになると思いましょう。

```ts
{ [K in keyof 型1]: 型2 }
```

ポイントは、こうすることでこのmapped typesは`型1`が指すオブジェクト型の各プロパティを引き継いだ新しい型を作るものであると明示されるということです。

Homomorphic mapped typeは、普通のmapped typeにはない特別な挙動をします。筆者が忘れているものがなければ、特別な挙動は以下の3つにまとめられます。

- 元の型（上の例でいう`型1`）が持つmodifierを引き継ぐ。
- 元の型が配列型やタプル型ならば、homomorphic mapped typeの結果も配列型やタプル型となる。
- （`型1`が型引数のとき限定）union distributionする。

次の例で、homomorphic mapped typeならばmodifierを引き継ぐことが確かめられます。

```ts
// homomorphic mapped typeである
type HMT<T> = { [K in keyof T]: T[K][] };
type NotHMT<T> = { [K in Extract<keyof T, unknown>]: T[K][] };

type FooObj = { readonly foo: string };
// type A = { readonly foo: string[] }
type A = HMT<FooObj>;
// type B = { foo: string[] }
type B = NotHMT<FooObj>;
```

この例ではhomomorphic mapped typeである`HMT<T>`と、homomorphicではないmapped typeである`NotHMT<T>`を使っています。両者は`keyof T`か`Extract<keyof T, unknown>`かにおいて異なっていますが、これらの型計算は常に同じ結果になるはずです。しかし、mapped typesの中で使われた場合、後者のようにするとhomomorphicであると認識されなくなります。

この違いが`A`と`B`の違いを生んでいます。`A`にはhomomorphic mapped typeを使っているので、元の`FooObj`が持っている`readonly`が受け継がれています。その一方で、`B`では受け継がれていません。

次に、タプル型の例を見てみましょう。

```ts
type HMT<T> = { [K in keyof T]: T[K][] };
type NotHMT<T> = { [K in Extract<keyof T, unknown>]: T[K][] };

type Tuple = [string, number, boolean];
// type A = [string[], number[], boolean[]]
type A = HMT<Tuple>;
// type B = {
//    [x: number]: (string | number | boolean)[];
//    [Symbol.iterator]: (() => IterableIterator<string | number | boolean>)[];
//    （中略）
//    includes: ((searchElement: string | ... 1 more ... | boolean, fromIndex?: number | undefined) => boolean)[];
// }
type B = NotHMT<Tuple>;
```

先ほどと同じ`HMT`および`NotHMT`にタプル型`Tuple`を食わせると、HMTの場合は新たなタプル型（`A`）が得られています。タプル型の各要素に`T[K][]`という部分が適用されていることが分かります。

一方で、HMTではない場合は結果がただのオブジェクト型となります。そして、結果（`B`）は壮絶なことになっています。これはタプル型が持つ文字通り全てのプロパティが普通にマップされた型です。タプル型の値は配列なので、タプル型が持つプロパティというのは配列が持つ全てのプロパティやメソッドになります。上の例では`[Symbol.iterator]`とか`includes`とかいったものが見えます。

配列やタプルの型の中身を加工したいという需要はそこそこあり、そのためにはhomomorphic mapped typesが必要です。

# TypeScript 4.5 Betaで明るみになった問題

この記事では、特にタプル型に対するhomomorphic mapped typesに注目します。皆さんはどんな活用法を思いつくでしょうか。

一つの例として、こんな例を考えてみましょう。

```ts
type PromiseContent<P> = P extends Promise<infer V> ? V : never;

type HMT<T extends readonly Promise<unknown>[]> = { [K in keyof T]: PromiseContent<T[K]> };

type Promises = [Promise<string>, Promise<string>, Promise<number>];
// type C = [string, string, number]
type C = HMT<Promises>;
```

なんと、homomorphic mapped typesを使うことで、Promiseが入ったタプル型から、Promiseの中身が入ったタプル型を作ることができました。そうなると、ひとつの活用法が思いつくはずです。そう、`Promise.all`ですね。`Promise.all`は渡されたPromiseの全てがfulfillされるまで待つ機能を持ち、その結果は配列で得られます。異なる結果の型を持つPromiseたちが渡されたときはそれらの型を返り値の型でも維持したいですから、タプル型の出番です。

`Promise.all`の型定義は、TypeScript 4.5 Betaでこのhomomorphic mapped typeを使ったものに書き換えられました。逆に言えば、従来の型定義はタプル型を使っていたものの、homomorphic mapped typeを使っていませんでした。

従来の型定義は次のようなものです（TypeScript 4.4のlib.es2015.promise.d.tsから引用。ただしコメントは除去）。

```ts
    all<T1, T2, T3, T4, T5, T6, T7, T8, T9, T10>(values: readonly [T1 | PromiseLike<T1>, T2 | PromiseLike<T2>, T3 | PromiseLike<T3>, T4 | PromiseLike<T4>, T5 | PromiseLike<T5>, T6 | PromiseLike<T6>, T7 | PromiseLike<T7>, T8 | PromiseLike<T8>, T9 | PromiseLike<T9>, T10 | PromiseLike<T10>]): Promise<[T1, T2, T3, T4, T5, T6, T7, T8, T9, T10]>;
    all<T1, T2, T3, T4, T5, T6, T7, T8, T9>(values: readonly [T1 | PromiseLike<T1>, T2 | PromiseLike<T2>, T3 | PromiseLike<T3>, T4 | PromiseLike<T4>, T5 | PromiseLike<T5>, T6 | PromiseLike<T6>, T7 | PromiseLike<T7>, T8 | PromiseLike<T8>, T9 | PromiseLike<T9>]): Promise<[T1, T2, T3, T4, T5, T6, T7, T8, T9]>;
    all<T1, T2, T3, T4, T5, T6, T7, T8>(values: readonly [T1 | PromiseLike<T1>, T2 | PromiseLike<T2>, T3 | PromiseLike<T3>, T4 | PromiseLike<T4>, T5 | PromiseLike<T5>, T6 | PromiseLike<T6>, T7 | PromiseLike<T7>, T8 | PromiseLike<T8>]): Promise<[T1, T2, T3, T4, T5, T6, T7, T8]>;
    all<T1, T2, T3, T4, T5, T6, T7>(values: readonly [T1 | PromiseLike<T1>, T2 | PromiseLike<T2>, T3 | PromiseLike<T3>, T4 | PromiseLike<T4>, T5 | PromiseLike<T5>, T6 | PromiseLike<T6>, T7 | PromiseLike<T7>]): Promise<[T1, T2, T3, T4, T5, T6, T7]>;
    all<T1, T2, T3, T4, T5, T6>(values: readonly [T1 | PromiseLike<T1>, T2 | PromiseLike<T2>, T3 | PromiseLike<T3>, T4 | PromiseLike<T4>, T5 | PromiseLike<T5>, T6 | PromiseLike<T6>]): Promise<[T1, T2, T3, T4, T5, T6]>;
    all<T1, T2, T3, T4, T5>(values: readonly [T1 | PromiseLike<T1>, T2 | PromiseLike<T2>, T3 | PromiseLike<T3>, T4 | PromiseLike<T4>, T5 | PromiseLike<T5>]): Promise<[T1, T2, T3, T4, T5]>;
    all<T1, T2, T3, T4>(values: readonly [T1 | PromiseLike<T1>, T2 | PromiseLike<T2>, T3 | PromiseLike<T3>, T4 | PromiseLike<T4>]): Promise<[T1, T2, T3, T4]>;
    all<T1, T2, T3>(values: readonly [T1 | PromiseLike<T1>, T2 | PromiseLike<T2>, T3 | PromiseLike<T3>]): Promise<[T1, T2, T3]>;
    all<T1, T2>(values: readonly [T1 | PromiseLike<T1>, T2 | PromiseLike<T2>]): Promise<[T1, T2]>;
    all<T>(values: readonly (T | PromiseLike<T>)[]): Promise<T[]>;
```

つまり、2〜10要素の場合はそれらに対応する個別のオーバーロードされたシグネチャを持たせていて、11要素以降は諦めていました（一番下のタプル型ではなく配列型を扱うシグネチャに入る）。

しかし、homomorphic mapped typesがあればこのような圧倒的な型定義は不要ですね。ということで、TypeScript 4.5 Betaでは次のようになりました。

```ts
  all<T extends readonly unknown[] | []>(values: T): Promise<{ -readonly [P in keyof T]: Awaited<T[P]> }>;
```

この1行だけです。技術の進歩は素晴らしいですね。返り値の中でhomomorphic mapped typeが使われています。`Awaited`というのが出てきていますが、[これもTypeScript 4.5 Betaで標準ライブラリに加えられた定義です](https://github.com/microsoft/TypeScript/pull/45350)。`Awaited`追加のおまけとして`Promise.all`などの型定義を改善したという雰囲気です。

ちなみに、型引数の`extends readonly unknown[] | []`という制約は、配列リテラルが引数として渡された場合は`T`を配列型ではなくタプル型に推論せよというおまじないです。`| []`の代わりに、`values: T`を`values: [...T]`とするというおまじないも可能です。好きなほうを使いましょう。

めでたしめでたし……と思いきや、ひとつ問題が起こりました。そう、タイトルにもある`any`などという問題児です。次のissueで報告されているように、`Promise.all`に`any`型の値を渡した際の挙動が従来と大きく異なるものになってしまいました。

https://github.com/microsoft/TypeScript/issues/46169

```ts
const nazo: any = 123;
// TypeScript 4.4 では any[] 型
const res = await Promise.all(nazo);
```

`res`はTypeScript 4.4では`[unknown, unknown, unknown, unknown, unknown, unknown, unknown, unknown, unknown, unknown]`型でした。それもどうなのと思いつつ、`Promise.all`が返す値が配列であるという情報が残っています。一方で、TypeScript 4.5 Betaでは`{ [x: string]: any; }`型となります。これは`any`型を上記のhomomorphic mapped typeに通した結果です。型引数に`T extends readonly unknown[] | []`という制約があるので`T`は必ず配列型あるいはタプル型であると思いきや、実は`any`が制約をすり抜けて`T`に入ってきてしまい、その結果としてhomomorphic mapped typeが配列ではないオブジェクト型を生成してしまいました。これにより、`res`が配列型であることに依存しているコードが壊れてしまったわけです。これがTypeScript 4.5 Betaで発生した問題です。

# 解決策

さて、冒頭で言及したDesign Meeting Notesではこの問題の解決策について議論されていることが分かります。すでにPull Requestも出されています。

https://github.com/microsoft/TypeScript/pull/46218

今回のhomomorphic mapped typeは`keyof T`の形であり、しかも`T extends readonly unknown[] | []`という制約の情報があります。そこでこのPRでは、homomorphic mapped typeに`any`が来てもその`any`が入っている型変数の制約から配列が意図していると判明するのであれば、homomorphic mapped typeの結果を配列型（`any[]`）にします。

Design Meeting Notesを見た感じでは、この方向性に対する異論はあまり無さそうに見えました。しかし、細かな議論は色々とあるようです。例えば、次のように、単なる配列型ではなく制約でタプル型の要素数まで判明している場合はそれを結果に反映すべきかどうかです。今の実装では反映されませんが、Notesに結論が載っていなかったのでどうなるかは未知数です。「どうせ`any`とか渡してる時点で型の信頼性無いんだし返り値の要素数までこだわらなくて良くない？（超意訳）[^note_1]」という意見も記録されていました。

[^note_1]: 原文: If an `any` flows in, should you really say the output has the right arity?

```ts
type F<T extends [unknown, unknown]> = { [K in keyof T]: T[K][] };

// 今のPRの実装では type A = any[][] になる
type A = F<any>;
```

また、次のような例も議論に上がっていました。PRの中にもこのようなテストケースが用意されていることから、こだわりがあるのかもしれません。次の`IndirectArrayish`は、このPRをもってしてもまだ結果が配列型にならない例です。

```ts
type Objectish<T extends unknown> = { [K in keyof T]: T[K] };

type IndirectArrayish<U extends unknown[]> = Objectish<U>;
// type A = { [x: string]: any; }
type A = IndirectArrayish<any>;
```

この例では、`Objectish`に`any`を渡しても返り値は当然ながら配列型にはなりません。`T extends unknown`であって、配列型を制約としていないからです。では、`IndirectArrayish`を経由した場合はどうでしょうか。この場合、`U extends unknown[]`という制約があるので、それを`Objectish<U>`の評価の際に使用すれば配列型を返せるように思えます。しかし、現在の仕様では`IndirectArrayish`の制約であり`U extends unknown[]`は`IndirectArrayish`の定義内でのみ有効であり、`Objectish`の中を評価する際には引き継がれません。そのため、`IndirectArrayish`を使った場合も結果は配列型にはならないのです。

もし`IndirectArrayish<any>`の結果を配列型にしたいのであれば、型引数の制約を伝播させる必要があります。個人的にはそこまでしなくて良いと思いますが。制約を伝播させるかどうかについての結論は“let's take it offline”と書いてあったのでこちらも不明です。

# まとめ

結果を配列型にすることを意図してhomomorphic mapped typesを仕様する場合、`any`が入ってきた場合に意図しない挙動となる恐れがあります。さすが問題児ですね。しかし、TypeScript 4.5ではその場合も`any[]`のような配列型を返すようにすることで、より安心してhomomorphic mapped typesを使えるようになることが期待されます。楽しみですね。

細かい仕様を追いたい方は先ほどのPRをウォッチしましょう。
