---
title: "イテレータを分岐させるとどうなる？　Iterator Helpersに見るJavaScriptのイテレータの挙動"
emoji: "🚪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "ecmascript"]
published: true
---

2024年10月のTC39ミーティングでは、[Iterator Helpers](https://github.com/tc39/proposal-iterator-helpers)がStage 4となり、ECMAScriptの仕様に追加されることが決定されました。Iterator HelpersはすでにGoogle Chromeなどで試すことができます。

Iterator Helpersは概してわかりやすい機能群ではありますが、やはり元々がJavaScriptということで、直観的には理解しがたい挙動もあります。そのような挙動は、とくにイテレータを分岐させたときに見られます。

ということで、この記事ではイテレータを分岐させた場合の挙動を見ていきましょう。

## イテレータを分岐させる

Iterator Helpersは、イテレータに生えたメソッドであり、返り値は新しく作られたイテレータです。そのため、同じイテレータメソッドに対して複数回ヘルパーメソッドを呼び出すと、同じイテレータを親とする複数のイテレータが得られることになります。この記事ではこのことを**イテレータの分岐**と呼びます。

```js
// 1から10までのイテレータ
const iter = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].values();

// イテレータを分岐させる
const sub1 = iter.map(x => x * 10);
const sub2 = iter.map(x => x * 100);
```

こうすると、sub1とsub2は同じ元のイテレータiterを親として持ちます。これらのイテレータがどんな挙動をするのか見てみましょう。

まず、sub1とsub2を交互に進めてみましょう。

```js
for (let i = 0; i < 5; i++) {
  console.log(sub1.next().value);
  console.log(sub2.next().value);
}
```

このコードを実行して得られる出力を予想してみてください。

正解は次の通りです。

:::details 出力
```js
10
200
30
400
50
600
70
800
90
1000
```
:::

いかがでしょうか？　筆者としては、わりと直観的な挙動だと感じます。イテレータの基本的な特徴は遅延評価であり、例えばsub1から値を取り出そうとしたときは、sub1はその場で親のiterから値を取り出し、それに自身の変換関数 `x => x * 10` を適用して値を出力します。sub2も同様です。

この例では、sub1がiterから1を取り出し、次にsub2がiterから2を取り出し、……のように処理が進んでいることが分かります。

値を全部取り出したときの挙動も見てみましょう。

```js
const iter = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].values();
const sub1 = iter.map(x => x * 10);
const sub2 = iter.map(x => x * 100);

for (const x of sub1) {
  console.log("sub1", x);
}

for (const x of sub2) {
  console.log("sub2", x);
}
```

このコードは、sub1の要素を全部取り出してから、次にsub2の要素を全部取り出します。出力を予想してみてください。

:::details 出力
```js
sub1 10
sub1 20
sub1 30
sub1 40
sub1 50
sub1 60
sub1 70
sub1 80
sub1 90
sub1 100
```
:::

この場合、sub1の値が無くなるときというのは、親であるiterの値が無くなったときです。よって、sub1の値を全部取り出しつくした時点でiterには値が残っていません。そのため、sub2の要素を取り出そうとしても1つも得られないことになります。

これは、`toArray`のような方法で値を全部取り出したときも同じです。

```js
const arr1 = sub1.toArray(); // [10, 20, 30, 40, 50, 60, 70, 80, 90, 100]
const arr2 = sub2.toArray(); // []
```

### filterの挙動

`map`メソッドの場合は、子イテレータから取り出される値は親イテレータから取り出される値と1対1で対応しています。しかし、Iterator Helpersのメソッドは全部がそうではありません。`filter`もその例です。

```js
const iter = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].values();
const sub1 = iter.map(x => x * 10);
const sub2 = iter.filter(x => x >= 5);

for (let i = 0; i < 5; i++) {
  console.log(sub1.next().value);
  console.log(sub2.next().value);
}
```

このコードの出力を正確に予測できたら、イテレータに対する理解度がなかなか高いと言えるでしょう。

:::details 出力
```js
10
5
60
7
80
9
100
undefined
undefined
undefined
```
:::

この出力は、sub2から最初に値を取り出した時点で、iterの2, 3, 4, 5が消費されたことを意味しています。`filter`メソッドが作ったイテレータは、このように条件に合致する値が得られるまで親イテレータから値を取り出し続けます。

### takeの挙動

`take`は、親イテレータから指定した数だけ値を取り出して、それで終了するイテレータを作ります。

```js
const iter = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].values();
const sub1 = iter.take(5);
const sub2 = iter.take(5);

const arr1 = sub1.toArray();
const arr2 = sub2.toArray();

console.log("arr1", arr1);
console.log("arr2", arr2);
```

この場合、arr1とarr2はどうなるか考えてみましょう。

:::details 出力
```js
arr1 [1, 2, 3, 4, 5]
arr2 [6, 7, 8, 9, 10]
```
:::

そう、この場合はarr1とarr2に5個ずつ値が入ります。この挙動は次のように説明できます。

まず、`sub1.toArray()`は、sub1から全ての値を取り出して配列に詰めようとします。sub1はiterから5個の値を（より正確には5個を上限として）取り出すイテレータです。そのため、5個取り出された時点でsub1は終了します。この時点でiterにはまだ5個の値が残っています。そのため、sub2はiterから残りの5個の値を取り出すことができます。

### dropの挙動

`drop`は、親イテレータの先頭から指定した数だけ値を捨てて、残りを順番に返すイテレータを作ります。

```js
const iter = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].values();
const sub1 = iter.map(x => x * 10);
const sub2 = iter.drop(5);

for (let i = 0; i < 5; i++) {
  console.log(sub1.next().value);
  console.log(sub2.next().value);
}
```

このコードの出力を予想してみましょう。もう慣れたものですね。

:::details 出力
```js
10
7
80
9
100
undefined
undefined
undefined
undefined
undefined
```
:::

この挙動は次のように説明できます。

sub2から最初に値を取り出そうとしたときに、親（iter）から5個の値を捨てる挙動が発生します。そのため、iterから取り出された2, 3, 4, 5, 6が捨てられ、sub2からは最初に7が出力されます。それ以降はsub2はただiterから1個ずつ値を取り出すだけです。

このように、Iterator Helpersはイテレータの処理を宣言的に書くことを助けてくれる一方で、分岐させるなど変なことをすると、裏でやっているわりと泥臭い処理が透けて見えるような挙動になります。

もちろん、そのようなことを推奨するわけではありません。しかし、知識として持っておくのは悪いことではありませんね。

いかがだったでしょうか。この記事ではイテレータクイズを通してJavaScriptのIterator Helpersの挙動を見てきました。次の記事もよろしくお願いします。

## ジェネレータとIterator Helpers

……と言いたいところですが、実はこの記事はここからが本題です。JavaScriptにはイテレータと密接に関わる**ジェネレータ**という機能があります。ジェネレータは、自前のイテレータで実装するための機能です。ジェネレータについても調べないとJavaScriptのイテレータを真に理解したとは言えません。

まず、ジェネレータ関数を使って無限イテレータを作ってみます。

```js
function* naturals() {
  let n = 1;
  while (true) {
    yield n;
    n++;
  }
}
```

このジェネレータ関数は、1から順番に値を返すイテレータを作ります。このイテレータは無限に値を返すため、`for..of`で全部取り出すことはできません。しかし、`take`を使って最初の5個だけ取り出すことはできます。

```js
const iter = naturals();
const sub1 = iter.take(5);

const arr1 = sub1.toArray(); // [1, 2, 3, 4, 5]
```

いいですね。分岐させるのも試してみましょう。

```js
const iter = naturals();
const sub1 = iter.map(x => x * 10);
const sub2 = iter.filter(x => x >= 5);

for (let i = 0; i < 5; i++) {
  console.log(sub1.next().value);
  console.log(sub2.next().value);
}
```

:::details 出力
```js
10
5
60
7
80
9
100
11
120
13
```
:::

先ほど学習した通りですね。今回は無限イテレータなので11以降の値も取り出せます。

では、takeで分岐させるのもやってみましょう。

```js
const iter = naturals();
const sub1 = iter.take(5);
const sub2 = iter.take(5);

const arr1 = sub1.toArray();
const arr2 = sub2.toArray();

console.log("arr1", arr1);
console.log("arr2", arr2);
```

:::details 出力
```js
arr1 [1, 2, 3, 4, 5]
arr2 []
```
:::

あれ？🤔

これは予想と違ったという人も多いのではないでしょうか。直観的にはarr2は`[6, 7, 8, 9, 10]`になりそうに思えます。しかし、実際には空配列になります。この理由を説明できるようになるのがこの記事のもうひとつの目標です。

## イテレータを閉じる

上記のような処理となる理由を一言で説明すると、「`take`で作られたイテレータは、規定回数の要素を取り出し終わったら親のイテレータを**閉じる**から」です。

この「閉じる」という概念は、ジェネレータに関わるものです。ジェネレータ（ジェネレータ関数が返したイテレータのことをジェネレータと呼びます）を閉じた場合、ジェネレータ関数の実行を外から強制終了させることができます。

ジェネレータは、イテレータが通常備える`next`メソッドに加えて`return`と`throw`メソッドを持っています。`return`メソッドを呼び出すことが、ジェネレータを閉じることに相当します。

### ジェネレータを閉じてみる例

まず、ジェネレータを閉じる例を見てみましょう。先ほどの`naturals`をちょっと修正して、閉じられたことが分かるようにします。

```js
function* naturals() {
  try {
    let n = 1;
    while (true) {
      yield n;
      n++;
    }
  } finally {
    console.log("naturals is closed");
  }
}

const iter = naturals();

for (let i = 0; i < 5; i++) {
  console.log(iter.next().value);
}

iter.return();
```

:::details 出力
```js
1
2
3
4
5
naturals is closed
```
:::

このコードでは、nextを5回呼び出したあとに`iter.return()`を呼び出しています。そうすると、`naturals is closed`と表示されます。つまり、ジェネレータ関数内で、謎の力で`while`ループを抜け出して関数が終了したことが分かります。

仕様書的な機序としては、`return`メソッドを呼び出した場合、ジェネレータ関数内のyield式でReturn Completionが発生します。分かりやすく言えば、ジェネレータ関数内のyield式がreturn文に化けます。よって、関数がその場で終了します。try-finally文におけるfinallyブロックはreturn文で関数から抜け出す場合にも忘れずに実行されるため、`naturals is closed`が表示されるわけです。

### takeとジェネレータ

ジェネレータを閉じるの意味が分かったところで、先ほどの例に立ち返ります。

```js
const iter = naturals();
const sub1 = iter.take(5);
const sub2 = iter.take(5);

const arr1 = sub1.toArray();
const arr2 = sub2.toArray();

console.log("arr1", arr1); // [1, 2, 3, 4, 5]
console.log("arr2", arr2); // []
```

`take(5)`で作られたイテレータは、5回は通常通り親イテレータから値を取り出して返します。6個目の値を取り出そうとした時点で、親イテレータを閉じます。`toArray()`はイテレータが終了するまで値を取り出し続けるため、`sub1.toArray()`を実行することでiterは閉じられます。

そのため、sub2は最初iterから値を取り出そうとしますが、iterはすでに閉じられているため何も取り出せません。そのため、sub2は1個も値を出力しないまま終了となり、結果として`sub2.toArray()`は空配列を返すことになります。

以上で、この場合の挙動も理解できましたね。なお、ジェネレータ以外のイテレータは基本的に`return`メソッドを持たないため、閉じられません（正確には、閉じようとしても何も起こりません）。そのため、このような挙動はジェネレータに特有のものです。

### イテレータを閉じる他のメソッド

`take`が親イテレータを閉じる場合があることが分かりました。他にも、親イテレータを閉じるメソッドがあります。takeも含めて以下の4つです。

- take
- some
- every
- find

`some`は、イテレータの値に対して命題関数を評価して、1つでも真なものがあればtrueを返します。親イテレータから1つずつ値を取り出して評価し、結果がtrueに確定した時点で親イテレータを閉じます。

```js
const iter = naturals();
const result = iter.some(x => x >= 5);

console.log(result); // true

const arr = iter.toArray();
console.log(arr); // []
```

この例では、iterから5以上の値が取り出されるまでiterから値が取り出され続けます。5が取り出されると、iterは閉じられ、`some`はtrueを返します。その後、iterから値を取り出そうとしても何も得られないため、`iter.toArray()`は空配列を返します。

`every`や`find`も同様に親イテレータに対するループを途中で打ち切る可能性があるメソッドです。その場合にやはり親イテレータが閉じられます。

他にも、コールバック関数を持つメソッドは、コールバック関数でエラーが発生した場合は親イテレータが閉じられます。このため、特殊な場合では`map`や`filter`なども親イテレータを閉じる可能性があります。

```js
const iter = naturals();
const sub1 = iter.map(x => {
  if (x >= 5) {
    throw new Error("x is too large");
  }
  return x * 10;
});

try {
  for (let i = 0; i < 10; i++) {
    console.log(sub1.next().value);
  }
} catch (e) {
  console.error(e.message); // x is too large

  const arr = iter.toArray();
  console.log(arr); // []
}
```

### Iterator Helpersから返されたイテレータを閉じる

実は、Iterator Helpersから返されたイテレータは`return`メソッドを持ちます。これを呼び出すと、親イテレータを閉じつつ、そのイテレータ自身も終了します。

```js
const iter = naturals();
const sub1 = iter.map(x => x * 10);

for (let i = 0; i < 5; i++) {
  console.log(sub1.next().value);
}

sub1.return(); // sub1が閉じられ、iterも閉じられる

console.log(sub1.next().value); // undefined
```

親が閉じられないイテレータだとしても、自分自身は閉じられると終了します。

```js
const iter = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].values();

const sub1 = iter.map(x => x * 10);
const sub2 = iter.map(x => x * 100);

sub1.return(); // sub1が閉じられるが、iterは閉じられない

console.log(sub1.next().value); // undefined
console.log(sub2.next().value); // 100
```

このように、Iterator Helpersから返されたイテレータは、ジェネレータと同様に閉じられるという性質を持っています。

## なぜジェネレータを閉じるのか

この記事で紹介した、ジェネレータに特有の閉じるという概念は、なぜ存在するのでしょうか。その理由は、ジェネレータがリソースを確保している場合に、リソースを解放するためだと考えられます。

ジェネレータは、関数を実行しながら値を生成する仕組みです。そのため、ジェネレータから値が取り出される可能性がある間は、関数の実行コンテキストを保持する必要があります。関数内で作られた変数や、その他確保されたリソースは解放されません。

ジェネレータが閉じられた場合、もうそのジェネレータ関数が実行再開することはありません。そのため、ジェネレータ関数内で確保されたリソースは解放可能になります。ジェネレータオブジェクト（これまでの例のiterなど）がまだ生きていた場合でも、閉じられた場合にはその裏の実行コンテキスト等はGC可能になります。

この記事でやったようにイテレータを分岐させてしまうと、閉じるという挙動が不自然な動きに繋がることもあります。しかし、そのようなユースケースはあまり無いだろうからリソース効率のほうを優先したのだろうと想像できます。

## まとめ

この記事では、Iterator Helpersを使ったときのイテレータの挙動について見てきました。特に、イテレータを分岐させた場合の挙動は、普段やることは無いでしょうが、理解できれば面白いですね。

記事の後半では、イテレータを閉じるという概念についても紹介し、Iterator Helpersとの関係を解説しました。

