---
title: "PromiseのUnhandled Rejectionを完全に理解する"
emoji: "🙅‍♂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "ecmascript"]
published: true
---

最近リリースされたNode.js 15ではデフォルトの設定が変更され、**Unhandled Rejection**が発生した際にプロセスが強制終了されるようになりました。

では、**Unhandled Rejection**がいつ発生するのか正確に説明できますか？　この記事では、Unhandled Rejectionに対する正確な理解を目指します。

## ECMAScript仕様書で確かめる

こういう場合に頼りになる唯一の情報源は**ECMAScript仕様書**、つまりJavaScriptの言語仕様を定める文書です。この記事では[ES2020の仕様書](https://tc39.es/ecma262/2020/)を参照します。

仕様書を"unhandled"で全文検索すれば、目的の記述を見つけるのはそう難しいことではありません。それは[25.6.1.9 HostPromiseRejectionTracker](https://tc39.es/ecma262/2020/#sec-host-promise-rejection-tracker)です。

これは抽象操作 (abstract operation) です。抽象操作とは仕様書内関数のようなもので、仕様書内の他の場所から呼び出されます。HostPromiseRejectionTracketの説明から最初の一文を引用します。

> HostPromiseRejectionTracker is an implementation-defined abstract operation that allows host environments to track promise rejections.

これは名前にHostと入っていることからも分かるように、具体的な内容は実行環境 (host environment) によって決められています。これから詳細を見ていきますが、HostPromiseRejectionTrackerはUnhandled Rejectionが発生したときに呼び出される抽象操作です。Node.js v15以降（デフォルトの設定）の場合は、HostPromiseRejectionTrackerが呼び出されたらプロセスが終了するというわけです。

ということで、次はHostPromiseRejectionTrackerがいつ呼び出されるのかを調べましょう。

## HostPromiseRejectionTrackerが呼び出される条件

仕様書内からHostPromiseRejectionTrackerの呼び出しを探すと、[25.6.1.7 RejectPromise](https://tc39.es/ecma262/2020/#sec-rejectpromise)が見つかります[^note_handled]。

[^note_handled]: もう一つ[25.6.5.4.1 PerformPromiseThen](https://tc39.es/ecma262/2020/#sec-performpromisethen)も見つかりますが、こちらは引数が違う（`"reject"`ではなく`"handle"`）ので今回はあまり関係ありません。これは少しあとで取り扱います。

このRejectPromiseという抽象操作の内容を引用します。

> When the RejectPromise abstract operation is called with arguments promise and reason, the following steps are taken:
>
> 1. Assert: The value of promise.[[PromiseState]] is pending.
> 2. Let reactions be promise.[[PromiseRejectReactions]].
> 3. Set promise.[[PromiseResult]] to reason.
> 4. Set promise.[[PromiseFulfillReactions]] to undefined.
> 5. Set promise.[[PromiseRejectReactions]] to undefined.
> 6. Set promise.[[PromiseState]] to rejected.
> 7. If promise.[[PromiseIsHandled]] is false, perform HostPromiseRejectionTracker(promise, "reject").
> 8. Return TriggerPromiseReactions(reactions, reason).

名前から察せられる通り、RejectPromise抽象操作はPromiseがrejectされるときに呼び出されるもので、この抽象操作の大部分はpromiseの状態変更です。例えば、ステップ6ではpromiseの[[PromiseState]]という内部スロットをrejectedに変更していますね。Promiseがrejectしたときはrejectされたときのコールバック（`catch`メソッドで登録されたものなど）が呼び出されるはずですが、それはステップ8で呼び出されているTriggerPromiseReactions抽象操作が担当しています。

さて、今回注目したいのはステップ7です。promiseの[[PromiseIsHandled]]フラグがfalseのとき、問題のHostPromiseRejectionTrackerが引数promiseと`"reject"`で呼び出されています。ですから、**[[PromiseIsHandled]]がfalseであるPromiseがrejectされたとき**がUnhandled Rejectionの発生条件となります。

では、Promiseの[[PromiseIsHandled]]が何を表しているのでしょうか。これが次に解明すべきことです。

## [[PromiseIsHandled]]の謎

まず、[[PromiseIsHandled]]はPromiseオブジェクトの内部スロットです。今更ですが、内部スロットというのは仕様書内から参照・操作可能なプロパティのようなものです。Promiseの内部スロットは各Promiseオブジェクトに結びついています。

ということで、Promiseオブジェクトが新規に作られたときに[[PromiseIsHandled]]がどうなっているか調べましょう。それは`new Promise`の仕様（[25.6.3 The Promise Constructor](https://tc39.es/ecma262/2020/#sec-promise-constructor)）を見れば分かります。

> 7​. Set promise.[[PromiseIsHandled]] to false.

とありますから、Promiseオブジェクトの[[PromiseIsHandled]]の初期値はfalseであることが分かります。

この[[PromiseIsHandled]]が変更されるのは仕様書上で1箇所だけです。具体的には、[25.6.5.4.1 PerformPromiseThen](https://tc39.es/ecma262/2020/#sec-performpromisethen)のステップ11です。

> 11​. Set promise.[[PromiseIsHandled]] to true.

PerformPromiseThenというのはどういう抽象操作なのか、何となく名前から想像がつきますね。Promiseに対して`then`メソッドが呼び出されたときに実行される抽象操作です。

ここまでをまとめると、Promiseオブジェクトに対して`then`を呼び出すとそのPromiseオブジェクトの[[PromiseIsHandled]]はtrueになります（[25.6.5.4 Promise.prototype.then](https://tc39.es/ecma262/2020/#sec-promise.prototype.then)）。

```js
const p = new Promise(()=> {});
// ここでは p.[[PromiseIsHandled]]はfalse

p.then(() => {});
// ここでは p.[[PromiseIsHandled]]はtrue
```

ちなみに、`catch`や`finally`を呼び出した場合も、内部的には`then`が呼び出されています。よって、Promiseの`catch`や`finally`を呼び出した場合も[[PromiseIsHandled]]はやはりtrueになることが分かります。

整理すると、**[[PromiseIsHandled]]がfalseのPromise**とは、**まだ`then`や`catch`, `finally`によってコールバック関数が登録されていないPromise**を指すことになります。

### awaitの扱い

ところで、最近はPromiseに対して`then`などを使わず、`await`を使ってPromiseを扱うことも多いですよね。その場合についても仕様に記述されています。この場合は内部的に上述のPerformPromiseThenが呼び出されます。

その一つが[6.2.3.1 Await](https://tc39.es/ecma262/2020/#await)で、これは仕様書の様々な場所から使用されています。次に引用するステップで、AwaitからPerformPromiseThenが呼び出されます。

> 9​. Perform ! PerformPromiseThen(promise, onFulfilled, onRejected).

代表例は`await`式です（[14.7.14 Runtime Semantics: Evaluation](https://tc39.es/ecma262/2020/#sec-async-function-definitions-runtime-semantics-evaluation)）。AwaitExpressionについての記述を引用します。

> *AwaitExpression*: `await` *UnaryExpression*
>
> 1. Let *exprRef* be the result of evaluating *UnaryExpression*.
> 2. Let *value* be ? GetValue(*exprRef*).
> 3. Return ? Await(*value*).

ステップ3でAwaitが使われています。これが意味することは、`await p`のようにPromiseを`await`すると、そのPromiseの[[PromiseIsHandled]]がtrueになるということです。他にどのような場合があるかはあとで列挙して説明します。

## Promiseはいつrejectされるか

ここまでで分かったことは、Promiseオブジェクトの[[PromiseIsHandled]]は、そのPromiseオブジェクトに対して`then`, `catch`, `finally`が呼ばれるか、またはそのPromiseオブジェクトが`await`された場合にtrueになるフラグだということです。

そして、このフラグがfalseである（＝まだ`then`などが呼ばれていない）Promiseがrejectした（RejectPromise抽象操作が呼ばれた）場合にHostPromiseRejectionTrackerが呼ばれる、すなわちUnhandle Rejectionとして扱われるということです。

次の疑問は、RejectPromiseが呼ばれるのは具体的にいつなのかということです。ここについては仕様にあまり深入りしませんので、きになる方は自分で調べてみましょう。

Promiseオブジェクトが作られる場合はそれに対応するresolve関数とreject関数が同時に作られます。resolve関数が呼ばれるとそのPromiseがresolveされ、reject関数が呼ばれるとそのPromiseがrejectされます。`new Promise`でPromiseを作るときにコールバック関数に渡される関数たちがまさにそれです。このうち`reject`関数が呼ばれたときが、まさにそのPromiseがrejectされるときとなります。例えば次のように作られたPromise `p`は、`p`に代入された段階ですでにrejectされています。

```js
const p = new Promise((resolve, reject) => {
  reject("hi");
});
```

他にも`then`のコールバック関数の中でエラーが発生した場合などもありますが、この記事ではそこは深追いしません。

### node.jsで試してみる

上の例をnode.jsで試してみましょう。次のコードをnode.js v15で実行してみます。
Promise `p`は[[PromiseIsHandled]]がfalseですから、`p`がrejectするとUnhandled Rejectionの条件を満たします（HostPromiseRejectionTrackerが呼び出されます）。

```js
const p = new Promise((resolve, reject) => {
    reject("hi");
});

console.log("wow");
```

実行してみると、このような出力が得られます。

```
wow
node:internal/process/promises:218
          triggerUncaughtException(err, true /* fromPromise */);
          ^

[UnhandledPromiseRejection: This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). The promise rejected with the reason "hi".] {
  code: 'ERR_UNHANDLED_REJECTION'
}
```

つまり、コードの最後（`console.log("wow")`）まで実行されてからUnhandled Rejectionをnode.jsが検知したことが分かります。

仕様上、HostPromiseRejectionTrackerはPromiseがrejectした瞬間に同期的に呼び出されます。つまり、`console.log("wow")`が呼び出されるよりも前にHostPromiseRejectionTrackerは実行されているはずです。それにも関わらず、node.jsは今実行中のコードを実行し終わるまでプロセスの終了を遅延したことになります（言い方を変えれば、プロセスの終了を次のイベントループで行なった）ことになります。

これは何故なのでしょうか、というのが次の話題です。

## プロセス終了キャンセル

何故なのかという問いに対する答えは、一言で言えばそうしないと使い物にならないからです。

次のように、`p.catch`の呼び出しを追加したコードを考えてみてください。

```js
const p = new Promise((resolve, reject) => {
    reject("hi");
});
p.catch(() => {
    console.log("omg!");
})

console.log("wow");
```

このコードでは`p`がrejectしますが、ではUnhandled Rejectionでプロセスが終了すべきでしょうか？　答えは否ですね。なぜなら、`p.catch`でちゃんとハンドラを追加しているからです。

実際、このコードをnode.jsで実行してもUnhandled Rejectionのエラーは発生せず、次のような出力で正常終了します。

```
wow
omg!
```

ところが、これはすこし不思議です。というのも、先ほどの説明だと`p`が作られた瞬間にHostPromiseRejectionTrackerがもう呼び出されているはずです。それにも関わらず、node.jsはUnhandled Rejectionエラーを発生させませんでした。

その理由も仕様に定められています。実は、一度HostPromiseRejectionTrackerが発動しても、直後にそのPromiseにハンドラを登録すればセーフなのです。これは、先述のPerformPromiseThenのステップ10.cに記述されています。

> If promise.[[PromiseIsHandled]] is false, perform HostPromiseRejectionTracker(promise, "handle").

これは、[[PromiseIsHandled]]なPromiseオブジェクトに対して`then`などでハンドラが追加された場合はHostPromiseRejectionTrackerを引数promise, `"handle"`で呼び出すということを意味しています。

実は、HostPromiseRejectionTrackerは2種類の場合に呼び出されます。1つはすでに出てきた引数`"reject"`の場合で、これは[[PromiseIsHandled]]がfalseなPromiseがrejectされたときに呼び出されます。もう1つが引数`"handle"`の場合であり、これは[[PromiseIsHandled]]がfalseなPromiseにハンドラが登録されたときに呼び出されます。

つまり、先ほどのコードでは`p`に対して`"reject"`でHostPromiseRejectionTrackerが呼び出された直後に、`"handle"`でHostPromiseRejectionTrackerが呼び出されていたのです。この場合node.jsはギリギリセーフとみなして、Unhandled Rejection例外を発生させません。このことは[node.jsのソースコード](https://github.com/nodejs/node/blob/master/lib/internal/process/promises.js)を見ると実際に書いてあります。

この機能により、一瞬でrejectされるPromiseを作ったとしても即座にハンドラを追加すればUnhandled Rejectionにはならないことが分かります。

ただし、node.jsはイベントループの次の回にプロセス終了をスケジュールするので、両者の間に間が開くとだめです。例えば、次のように`p.catch`を`setTimeout`のコールバックで行うようにすると間に合わずにUnhandled Rejectionが検知されます。

```js
const p = new Promise((resolve, reject) => {
    reject("hi");
});
setTimeout(() => {
    p.catch(() => {
        console.log("omg!");
    })
    console.log("wow");
}, 0);
```

実行結果：

```
node:internal/process/promises:218
          triggerUncaughtException(err, true /* fromPromise */);
          ^

[UnhandledPromiseRejection: This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). The promise rejected with the reason "hi".] {
  code: 'ERR_UNHANDLED_REJECTION'
}
```

一方で、`await null`や`process.nextTick`ならセーフとなります。

```js
const p = new Promise((resolve, reject) => {
    reject("hi");
});
await null;
process.nextTick(() => {
    p.catch(() => {
        console.log("omg!");
    })
    console.log("wow");
});
```

実行結果:

```
wow
omg!
```

尤も、このようにギリギリを攻めるのはなるべく避けるべきでしょう。Unhandled Rejectionでプロセスが終了するのを避けるためのベストプラクティスは、**Promiseを作ったら即座にハンドラを登録する**のが得策です。

結局いつまでにハンドラを登録すれば間に合うのかについては、**Promiseが作られた同期的な実行の最中に登録するのが確実**です。同期的な実行とは、`await`やコールバックによって分かれていない一連の実行のことです。JavaScriptでは同期的な実行に割り込んで何かが起こることはなく、それはUnhandled Rejectionも例外ではありません。コールバック関数などは今の同期的な実行が終了したあとで別に呼ばれることが多くあり、`await`も一旦そこで実行を終了（中断）します。これらは同期的な実行ではありません。

上で見たように、厳密には同期的な実行でなくても大丈夫なのですが（node.jsの場合`process.nextTick`など）、こういったものに頼るのはギリギリを攻めすぎなので避けるべきでしょう。

### Promiseチェーンに注意する

ところで、次のような場合には注意してください。

```js
const p = new Promise((resolve, reject) => {
    reject("hi");
});

p.then(() => {
    console.log("Hello");
});
```

これは`p`がrejectしますが、`p`にちゃんと`then`でコールバックを登録して[[PromiseIsHandled]]をtrueにしているのでOKでしょうか？　答えは否です。なぜなら、`p.then`の返り値が新しいPromiseだからです。

```js
const p = new Promise((resolve, reject) => {
    reject("hi");
});

// ↓このp2もrejectする
const p2 = p.then(() => {
    console.log("Hello");
});
```

`p.then`の返り値を`p2`とすると`p2`もPromiseです。しかも、`p`がrejectすると連鎖的に`p2`もrejectします。なぜなら、`p.then`で登録された関数は`p`がfulfill（成功）した場合にしか呼び出されず、rejectの場合はそのrejectをそのまま`p2`に受け継ぐからです。これにより、`p2`でUnhandled Rejectionが検知されます。

一方で、`p.catch`も新しいPromiseを返しますが、これは大丈夫です。

```js
const p = new Promise((resolve, reject) => {
    reject("hi");
});

const p2 = p.catch(() => {
    console.log("omg!");
});
```

この場合でも`p2`は新しいPromiseとなりますが、`p2`はrejectしません。なぜなら、`p`がrejectしたとき`catch`で登録したコールバック関数がエラーを処理してしまい、その場合も`p2`はfulfill（成功）となるからです。このように、`catch`はrejectするPromiseをfulfillするPromiseに変換するという役割を担っており、Unhandled Rejectionを防止する観点から重要です。

ただし、`catch`のコールバック関数内でエラーが発生した場合は`p2`もrejectされるので、`catch`内でエラーを発生させないように（あるいは発生する可能性がある場合はさらに別の`catch`を用意するように）注意しましょう。

```js
const p = new Promise((resolve, reject) => {
    reject("hi");
});
// ↓ p2がrejectしてしまう
const p2 = p.catch(() => {
    throw new Error("Ohh!");
});
```

## Promiseがawaitされる条件の整理

さて、Promiseの[[PromiseIsHandled]]がtrueになる条件について、先ほど後回しにしていたので整理しましょう。

### await式の場合

先ほども述べたように、`await p`とした場合は`p`の[[PromiseIsHandled]]がtrueになりますから、`p`がrejectしても即座にUnhandled Rejectionとなるわけではありません。

ただし、Promiseを作ったら`await p`を即座に実行しなければいけないことは変わりません。これは先ほど説明したように、`p.catch`を即座に登録しなければいけないのと同じことです。

また、`await p`の`p`がrejectしてもそれは完全に無視されるわけでなく、**例外**に変換されます。
次の例を見てみましょう。

```ts
async function main() {
  const p = new Promise((resolve, reject) => {
    reject("hi");
  });
  await p;
}

// p2がrejectしてしまうのでUnhandled Rejection発生
const p2 = main();
```

こうすると、`p`自体は`await p`されているのでUnhandled Rejectionの引き金とはなりませんが、`await p`で例外が発生したことにより`main`の実行が失敗となり、すなわち`main()`が返した`p2`がrejectされます。これを放置するとUnhandled Rejectionの引き金となります。

このように、`await`されたrejectは例外となって外側に伝播します。どこかで食い止めなければUnhandled Rejectionの引き金となるのは変わりません。これを防ぐには`main().catch(...)`のようにする方法が一つ、または`main`内で`try-catch`を使って例外をキャッチする必要があります。

### awaitのタイミング

基本的に、「Promiseを`await`すれば[[PromiseIsHandled]]扱いになる」と理解して差し支えありません。ただし、Promiseが`await`されるのは`await`式の場合に限るのではなく、自動的に`await`される場合も存在しています。仕様の用語で言えば、少し前に出てきた[6.2.3.1 Await](https://tc39.es/ecma262/2020/#await)が呼び出される場合です。ここではこのケースを列挙します。大きく分けて2つに分けられます。

#### Asyncイテレータ関係

Asyncイテレータを扱う場合、Promiseが自動的にawaitされることが多くあります。例えば、次の例では`Promise.reject`が2箇所で使用されており、以下のコードを実行するとどちらも実際に呼び出されます。

具体的には、`yield*`でasyncイテレータがイテレートされる際、asyncイテレータの`next`メソッドで返されたPromise、そのPromiseの結果のオブジェクトが持つ`value`プロパティに入っているPromise, `return`メソッドが返したPromise、また`throw`メソッドが返したPromiseが自動的にawaitされます（[14.4.14 Runtime Semantics: Evaluation](https://tc39.es/ecma262/2020/#sec-generator-function-definitions-runtime-semantics-evaluation)）。

```js
const asyncIter = {
  [Symbol.asyncIterator]: () => {
    return {
      next: () => Promise.resolve({
        done: false,
        value: Promise.reject("hey!"),
      }),
      return: () => Promise.reject("hi"),
    }
  },
}

async function* main() {
  yield* asyncIter;
}
main().next().catch(err => {
  console.log(err);
})
```

実行結果は次の通りです。最終的に`main().next()`が返したPromiseがrejectするので、それをcatchすればOKとなります。
`Promise.reject("hey!")`や`Promise.reject("hi")`で作られたrejectするPromiseは、内部的に`await`されるので問題ありません。

```
hi
```

また、同様にfor-await-ofでasyncイテレータをループさせた場合も`next()`の結果のPromiseや`return()`の結果のPromiseを自動的にawaitしますが、`next()`の結果のオブジェクトの`value`プロパティの中身のPromiseは自動的にawaitしないという違いがあるため気をつけてください。次の例は`main`関数の中身を変えただけですが、`Promise.reject("hey!")`が作られたままで放置されるためUnhandled Rejectionエラーとなります。

```js
const asyncIter = {
  [Symbol.asyncIterator]: () => {
    return {
      next: () => Promise.resolve({
        done: false,
        value: Promise.reject("hey!"),
      }),
      return: () => Promise.reject("hi"),
    }
  },
}

async function* main() {
  // yield* asyncIter;
  for await (const _ of asyncIter) {
    break;
  }
}
main().next().catch(err => {
  console.log(err);
})
```

実行結果:

```
hi
node:internal/process/promises:218
          triggerUncaughtException(err, true /* fromPromise */);
          ^

[UnhandledPromiseRejection: This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). The promise rejected with the reason "hey!".] {
  code: 'ERR_UNHANDLED_REJECTION'
}
```

ですから、あの位置に失敗する可能性があるPromiseを書くのはあまり勧められませんね。

#### Asyncジェネレータ関数関係

asyncジェネレータ関数は、asyncイテレータを作るための便利な構文です。asyncジェネレータ関数の処理にもいくつかの自動awaitが仕込まれています。例えば、asyncジェネレータ関数から`return`で返されたものは自動的にawaitされます（[13.10.1 Runtime Semantics: Evaluation](https://tc39.es/ecma262/2020/#sec-return-statement-runtime-semantics-evaluation)）。

```js
async function* g() {
  return Promise.reject("hey!");
}

g().next().catch(err => {
  console.log(err);
})
```

この場合、実行結果は次のようになります。これは、`g()`が返した`Promise.reject("hey!")`が`g().next()`が作ったPromise（これは別のPromiseです）の結果に伝播してrejectを発生させたことを意味しています。

```
hey!
```

同様に、asyncジェネレータ関数が`yield`したものや、逆に`yield`の返り値となったもの（外部から入力されたもの）も自動的にawaitされます（[25.5.3.7 AsyncGeneratorYield](https://tc39.es/ecma262/2020/#sec-asyncgeneratoryield）。

## まとめ

この記事では、最近話題のUnhandled Rejectionについて仕様の観点から解説しました。Unhandled Rejectionは、ハンドラが登録されていないしawaitもされていないPromiseがrejectしたときに発生するものです。

Promiseが即座にrejectされる可能性を考えると、Unhandled Rejectionを防ぐためには、得られたPromiseが失敗するかもしれないときは即座に`catch`などを呼び出すか`await`する必要があります。