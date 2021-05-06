---
title: "TypeScriptでexistential typeが欲しくなったときはカプセル化で我慢しよう"
emoji: "💊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

TypeScript でプログラミングをしていると、existential type （存在型）が欲しくなることがあります。そのような課題が発生した際は `any`や`as`を使って何とかしてしまいがちですが、実はある種のカプセル化を行うことでこれらの危険な機能を使わずに解決することができます。

## Existential Type が欲しくなる例

簡単な例として、こんなプログラムを書きたい場合を考えてみましょう。ここではまだ型は書いていません。

```js
function useNumber(num: number) {
  console.log(num);
}
function useString(str: string) {
  console.log(str);
}

const thunks = [
  [3, useNumber],
  ["foo", useString],
  [10, useNumber],
];

for (const [param, func] of thunks) {
  func(param);
}
```

注目すべきは変数`thunks`で、これは 2 要素タプルの配列になっています。各タプルは、「値」と「その値を受け取る関数」のペアとなっています。その後の for 文で実際に関数呼び出しを処理しています。つまり、配列`thunks`は予約された関数呼び出しの一覧であると見なすことができます。各関数はそれぞれ型が異なります。

このようなパターンはイベントディスパッチングのような処理を実装する際によく現れるので、読者の中には心当たりがある方が結構いるのではないかと思います。

### 型が付けられない！

実は、このプログラムに型を付けるのは困難です。変数`thunks`の型がどうなるのかちょっと考えてみましょう。TypeScript でできる範囲で考えると、タプルの左は何が来るか分からないので必然的に`unknown`になります（`any`などの型安全でない機能は一旦思考から除外しましょう）。そうすると右は「`unknown`を受け取る関数」なので`(arg: unknown) => void`のような型になります（今回とりあえず関数の返り値は`void`として進めます）。そうすると`thunks`の型は`[unknown, (arg: unknown) => void][]`と考えられますが、実はこれではうまくいきません。次に示すような型エラーが出てしまいます。

```ts
const thunks: [unknown, (arg: unknown) => void][] = [
  [3, useNumber],
  //  ^^^^^^^^^
  // エラー: Type '(num: number) => void' is not assignable to type '(arg: unknown) => void'.
  ["foo", useString],
  //      ^^^^^^^^^
  [10, useNumber],
  //   ^^^^^^^^^
];
```

エラーを見てみればこれは納得ですね。タプルの右は`(arg: unknown) => void`型と定義されているので、引数が`unknown`型、つまりどんな値の型でも受け入れる関数という意味になってしまっています。`useNumber`は`number`型しか受け取れないのでこれを満たしていません。

しかし、本来は`useNumber`は`number`が受け取れれば十分です。なぜなら、引数が`3`であることが分かっているからです。このような理屈は TypeScript の型では（今のところ）表現することができません。

もし existential type があれば、次のように型を書くことができたでしょう（existential type の構文は適当です）。

```ts
const thunks: (exists E. [E, (arg: E) => void])[]
```

つまり、配列の各要素が`(exists E. [E, (arg: E) => void])`型であり、これは「何らかの`E`型に対して`[E, (arg: E) => void]`である」という意味です。ポイントは、`E`は要素ごとに異なってもいいということです。上の例では、最初と 3 番目の要素では`E`が`number`であり、2 番目の要素では`E`が`string`であると考えれば辻褄が合います。Existential type があれば、このようなロジックを一発で表現することができます。

現実には existential type は TypeScript に存在しないので、`any`や`as`などを使わなければ上のロジックに型を付けることができません。

## カプセル化で代替する

さて、実は、少し妥協して**カプセル化**の考え方に基づいてプログラムを書き換えることで、`any`や`as`を使わずに型定義を行うことができます。具体的には、次のようにします。

```ts
function createThunk<E>(value: E, func: (arg: E) => void): () => void {
  return () => func(value);
}

const thunks: (() => void)[] = [
  createThunk(3, useNumber),
  createThunk("foo", useString),
  createThunk(10, useNumber),
];
for (const thunk of thunks) {
  thunk();
}
```

ランタイムで変わった点は、タプルではなく`createThunk`という関数に値と関数を渡した結果を配列に入れているという点です。`createThunk`の返り値は、型を見るとわかる通り`() => void`です。

また、for 文の中身が`thunk();`だけになっています。「値と関数のペアを受け取って値を関数に渡す」という処理は`createThunk`の中に移っています。

ここが**カプセル化**になっています。`E`が具体的に何型なのかという情報が`createThunk`内に秘匿され、関数の外には`() => void`のみが見えています。これにより`thunks`の型は`(() => void)[]`という単純なものになり、existential type のような機構が無くても型安全なプログラムになりました。

見方によっては、この書き換えは existential type を forall 型（総称型、いわゆるジェネリクス）に変換したとも捉えられます。また、関数呼び出しという構文的なアノテーションによって、型変数`E`を推論する機会を TypeScript コンパイラに与えています。このように、existential type が欲しくなった場合はジェネリクスで代替できることが多いでしょう。

## まとめ

TypeScript で existential type が欲しくなった場合は、設計を工夫してカプセル化することで型安全なままやりたいことを実現できる可能性があります。ここでは具体的な例を一つ見てきましたが、同様の考え方が通用する場面は色々とあります。

TypeScript の能力に合わせて適切な設計ができるようになれば、TypeScript マスターに一歩近づくことができるでしょう。
