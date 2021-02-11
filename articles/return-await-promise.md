---
title: "徹底解説！　return promiseとreturn await promiseの違い"
emoji: "⏳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "ECMAScript"]
published: true
---

先日、こちらのツイートを見かけました。

https://twitter.com/come25136_spd/status/1359134738829594624

それに対して筆者は以下のツイートをしたところ、いくつかの反応が寄せられました。

https://twitter.com/uhyo_/status/1359137289759121415?s=20

コード部分を再掲します。

```js
async function foo1() {
    return await Promise.resolve();
}
async function foo2() {
    return Promise.resolve();
}
async function wait() {
    await null;
}

// pika
// chu
// と表示される
foo1().then(() => console.log("pika"));
wait().then(() => console.log("chu"));

// chu
// pika
// と表示される
foo2().then(() => console.log("pika"));
wait().then(() => console.log("chu"));
```

関数`foo1`と`foo2`は、`Promise.resolve()`を`await`してから`return`するか、それとも`await`せずに`return`するかだけが違います。`async`関数はPromiseを`return`するとそのPromiseの結果が`async`関数の結果となるので、意味的にはどちらも同じはずです。しかし、コードの後半部分で`wait()`と競争させてみると分かるように、`foo2`は`foo1`よりも少し遅いようです。なお、あとで詳しく説明しますが、このコードの実行結果は言語仕様通りであり、常にこの順番となります。たまたま時間がかかって`pika`と`chu`の順番が入れ替わるというようなことは決して発生しません。`await`に何ミリ秒時間がかかるとかそういう話ではなく、言語仕様上の挙動の違いによって`foo1`と`foo2`の違いが発生しているのです。

この記事では、どうしてこのような挙動が発生するのかを全て解説します。なお、この記事ではGoogle ChromeのようなWebブラウザ上でコードを実行する場合を前提とします。

## Promiseとマイクロタスク

Promiseについて理解するに当たって欠かせないのが**マイクロタスク**の概念です。マイクロタスクは[HTML仕様書](https://html.spec.whatwg.org/multipage/webappapis.html#queue-a-microtask)に登場する概念です。簡単に言えば今同期的に実行中のプログラムが終了したあとに実行されるプログラムだと思って構いません。マイクロタスクはマイクロタスクキューに登録され、マイクロタスクが実行できるタイミングになったらキューから順番に取り出されて実行されます。

実は、Promiseの`then`メソッドのコールバックはマイクロタスクとして実行されます。そのことは、[ECMAScript仕様書 27.2.5.4.1 PerformPromiseThen](https://tc39.es/ecma262/#sec-performpromisethen)を見れば分かります。すでに状態がfulfilledであるPromiseに対して`then`メソッドを実行した場合、以下に引用する部分が実行されます。

> a. Let value be promise.[[PromiseResult]].
> b. Let fulfillJob be NewPromiseReactionJob(fulfillReaction, value).
> c. Perform HostEnqueuePromiseJob(fulfillJob.[[Job]], fulfillJob.[[Realm]]).

HostEnqueuePromiseJobは実行環境によって異なる定義がされる抽象操作であり、Webブラウザの場合は[HTML仕様書で次のように定義されています（一部抜粋）](https://html.spec.whatwg.org/multipage/webappapis.html#hostenqueuepromisejob)。確かにQueue a microtaskとありますね。

> 2​. Queue a microtask on the surrounding agent's event loop to perform the following steps:

実際に確かめてみましょう。`Promise.resolve`は[すでにfulfillされたPromiseを返します](https://tc39.es/ecma262/#sec-promise.resolve)。それに対して`then`を呼び出すと、`then`に渡されたコールバック関数はマイクロタスクとして登録され実行されます。

```js
console.log("1");
Promise.resolve().then(() => { console.log("2"); });
console.log("3");
Promise.resolve().then(() => { console.log("4"); });
console.log("5");
```

これを実行すると次の順でログが表示されます。

```
1
3
5
2
4
```

これにより、プログラムの一連の実行が終了してからマイクロタスクが順番に実行されていることが分かります。

## マイクロタスク時計を作る

`Promise.resolve`と`then`を利用して、マイクロタスクが一周するごとに1つカウントアップする時計を作ってみましょう。

```js
function clock() {
  let time = 0;
  rec();
  function rec() {
    console.log(time);
    Promise.resolve().then(() => {
      time++;
      if (time <= 5) rec();
    })
  }
}
```

試しに`clock()`を実行してみると次のようにログが出ます。

```
clock()
0
1
2
3
4
5
```

`0`を即座に表示し、カウントアップ処理をマイクロタスクとして登録するのを`5`まで繰り返しています。

`clock(); clock();`のように2つ動かすと、次のように表示されます。

```
0
0
1
1
2
2
3
3
4
4
5
5
```

このことから、2つの`clock()`が並行的に動作していることが分かります。2つの`clock`がそれぞれ`0`を表示し、各々マイクロタスクを登録します。マイクロタスクはキューに積まれるのでそれらが順番に`1`を表示し、結果として`1`が2つ表示されます。それ以降も同様です。

以降はこの`clock()`を時間の基準としましょう。試しに、`clock()`と`Promise.resolve().then`を並行に動作させてみます。

```js
clock();
Promise.resolve().then(() => console.log("pika!"));
```

```
0
1
pika!
2
3
4
```

このように表示されることから、`Promise.resolve().then`はマイクロタスク1回で`console.log`までたどり着くことが分かります。この“マイクロタスク1回”を1マイクロタスク（1mt）と呼ぶことにしましょう（この記事の造語です）。

1mtという時間単位は、非同期処理がマイクロタスクのみによって進行する場合にのみ有効です。他の要因での待ち時間が発生する場合はマイクロタスクで測ることはできなくなります。例えば`setTimeout`はマイクロタスクキューが全て消化されるまで待つので無限mt（？）となります。

```js
clock();
setTimeout(() => console.log("timeout"), 0);
// 0
// 1
// 2
// 3
// 4
// 5
// timeout
// と表示される
```

## `await null`の時間を測る

マイクロタスク時計を使用して、`async`関数の中で`await null`を実行した場合（Promiseでないものを`await`した場合）にかかる時間を計測してみましょう。

```js
async function run() {
  console.log("A");
  await null;
  console.log("B");
  await null;
  console.log("C");
}

clock();
run();
```

実行結果は次のようになります。

```
0
A
1
B
2
C
3
4
5
```

このことから、`async`関数は同期的に（時刻0mtで）実行され、`await`するたびに1mt経過することが分かります。つまり、Promise以外のものを`await`するのは1mtかかります。`Promise.resolve().then`と同じですね。ちなみに、すでにfulfillされたPromiseを`await`するのも同様に1mtかかります。

## 本題: `async`関数にかかる時間

では、ここからが本題です。`async`関数の実行にかかる時間を計測してみましょう。まずは何もしない`async`関数です。

```js
async function empty() {}

clock();
empty().then(() => console.log("done"));
```

結果はこうなります。

```
0
1
done
2
3
4
5
```

つまり、`empty()`は`Promise.resolve()`と同様にすでに解決されたPromiseを返すものと見なせます。実際、ECMAScript仕様書の[27.7.5.1 AsyncFunctionStart](https://tc39.es/ecma262/#sec-async-functions-abstract-operations-async-function-start)などを見ると、`async`関数は常にPromiseを返し、このように中で`await`しないものは即座に（同期的に）resolveすると書いてあります。このことは、上の`empty`関数は次のような関数と同等であることを意味しています。

```js
function empty() {
  return new Promise((resolve) => {
    resolve();
  })
}
```

これはまた、`Promise.resolve`の定義とほぼ同等です。このことから、空の`async`関数は解決までに0mtかかるものであることが分かります。それに対して`then`を呼び出すところで1mtかかるので、上の例で`"done"`が表示されるのは時刻1mtとなります。

では、いよいよ`foo1`の所要時間の話に進みます。

```js
async function foo1() {
    return await Promise.resolve();
}
```

これは、次の関数と同等です。

```js
async function foo1() {
  const res = await Promise.resolve();
  return res;
}
```

そして、`Promise.resolve()`を`await`した結果は`undefined`なので、これは結局次と同じです。

```js
async function foo1() {
  await Promise.resolve();
}
```

先ほどすでにfulfillされたPromiseを`await`するのは1mtかかると説明したので、この`foo1`の実行時間も1mtです。よって、`foo1().then(() => { console.log("pika") })`は`then`自身も1mt消費するので、時刻2mtに`"pika"`が表示されることが期待されます。試してみましょう。

```js
async function foo1() {
    return await Promise.resolve();
}

clock();
foo1().then(() => { console.log("pika") })
```

やってみると、こうなります。

```
0
1
2
pika
3
4
5
```

予想通り、時刻2mtに`pika`と表示されました。

次に`foo2`です。

```js
async function foo2() {
    return Promise.resolve();
}
```

これは中で`await`せずに`Promise.resolve()`を返すので、次の関数と同等です。

```js
function foo2() {
  return new Promise((resolve) => {
    resolve(Promise.resolve());
  })
}
```

Promiseの`resolve`関数にPromiseが渡されたときの挙動は、[27.2.1.3.2 Promise Resolve Functions](https://tc39.es/ecma262/#sec-promise-resolve-functions)に記述されています。それによれば、`resolve`関数に渡されたPromiseの`then`メソッドを呼ぶというタスクを**マイクロタスクとして実行します**（仕様書から一部抜粋）。

> 9​. Let then be Get(resolution, "then").
> 11. Let thenAction be then.[[Value]].
> 13. Let thenJobCallback be HostMakeJobCallback(thenAction).
> 14. Let job be NewPromiseResolveThenableJob(promise, resolution, thenJobCallback).
> 15. Perform HostEnqueuePromiseJob(job.[[Job]], job.[[Realm]]).

以上の仕様のステップ14に現れているjobというのが「渡されたPromiseの`then`メソッドを呼ぶ」というタスクであり、それをステップ15でマイクロタスクとして登録しています。

言い方を変えれば、Promiseの`resolve`関数にPromiseが渡された場合、1mt後にその`then`メソッドが呼ばれるのです。つまり、`foo2`は以下のように書き換えられます。

```js
function foo2() {
  return new Promise((resolve, reject) => {
    Promise.resolve().then(() => {
      Promise.resolve().then(resolve, reject);
    })
  })
}
```

見れば分かるように、`foo2()`が返すPromiseがfulfillされるまでに`Promise.resolve().then`が2回噛ませています。これにより、`foo2()`が返すPromiseは2mt後にfulfillされることになります。これは先ほどの`foo1()`より1mt多いですね。

このPromiseにさらに`then`を噛ませて`console.log("chu")`を実行するとなると、`chu`が表示されるのは時刻3mtです。

```js
clock();
foo2().then(() => console.log("chu"));
```

```
0
1
2
3
chu
4
5
```

このように、async関数からPromiseを返すという行為は、`await`とは異なり2mtかかるのです。

まとめとして、`foo1`と`foo2`を一緒に実行してみましょう。

```js
clock();
foo1().then(() => console.log("pika"));
foo2().then(() => console.log("chu"));
```

```
0
1
2
pika
3
chu
4
5
```

このように、`pika`と`chu`は実行される時刻が違います。
`foo1()`よりも`foo2()`のほうが1mtだけ長く時間がかかります。
その理由は、`async`関数の中の`await`は1mtかかり、Promiseでないものを返すのは0mtかかる一方で、Promiseをreturnするのは2mtかかるからです。

## まとめ

この記事では、以下の2つの関数の挙動の違いを説明しました。具体的には、`foo2`のほうが、返り値のPromiseが解決されるまでマイクロタスク1回分長くかかります。

```js
async function foo1() {
    return await Promise.resolve();
}
async function foo2() {
    return Promise.resolve();
}
```