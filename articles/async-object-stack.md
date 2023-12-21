---
title: "AsyncLocalStorageとusingで快適に構造化ロギングしたい話"
emoji: "🪒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nodejs", "javascript"]
published: true
---

アプリケーションのログ収集にあたっては、**構造化ロギング** (structured logging) というプラクティスが広く実践されています。構造化ロギングとは、ログの出力を単なる文字列ではなく、メッセージ以外のメタデータも含む構造化されたデータとして出力することです。構造化されたデータを出力することで、ログの解析や集計を容易にすることができます。

この記事では、JavaScriptのサーバーサイドアプリケーションにおける構造化ロギングの実装に焦点を当てて議論し、最終的に筆者が開発した`async-object-stack`を宣伝します。

https://github.com/uhyo/async-object-stack

## コンテキストをどのように共有するか

構造化ロギングの実装における主要な関心事は、複数のログでどのようにメタデータを共有するかです。ログに付与するメタデータは、1つのログだけでなく、複数のログにまたがって付与されることが多いでしょう。例えば、リクエストを送ってきたユーザーのIDが判明しているのであれば、その後全てのログにそのユーザーIDを付与したいでしょう。他にトレースID等も有用なメタデータです。

ログの出力は、プログラム内のあらゆる場所で行われる可能性があります。特に、実用的なプログラムはたくさんの関数から構成され、一つのリクエストの処理の間にも関数を出たり入ったりすることになります。

全ての場所で適切なメタデータを出力するためには、「現在のメタデータ」を持ち回ることになります。このように現在の状況を表すデータのことを**コンテキスト**と呼ぶことが多いので、この記事でもコンテキストと呼んでいきます。メタデータを持ち回る愚直な実装の一例を示します。

```js
async function handleRequest(request, metadata) {
  metadata = { ...metadata, requestID: request.id };
  const userId = await getUserId(request, metadata);
  if (userId === null) {
    log("User not found", metadata);
    return;
  }
  metadata = { ...metadata, userId };
  log("Processing request", metadata);
  await doSomething(metadata);
  await doSomethingElse(metadata);
  await doSomethingElseAgain(metadata);
  log("Finished processing request", metadata);
}
```

このように引数を持ち回るのは冗長に感じます。しかしながら、例えばグローバル変数にメタデータを保存しておくようなことはできません。非同期処理でなければグローバル変数でも大丈夫なのですが、非同期処理でそれをやってしまうと、複数のリクエストが同時に処理される場合にメタデータが混ざってしまいます。

この問題を解決するものとして、Node.jsには`AsyncLocalStorage`というAPIが実装されています。また、互換のAPIがDenoやBun、そしてCloudflare Workersにも実装されています。また、JavaScript本体においても類似のAPIが[Async Context for JavaScript](https://github.com/tc39/proposal-async-context)として議論されていますので、将来的にはこちらが推奨されるようになるかもしれません。

`AsyncLocalStorage`は、非同期処理を含むようなコールスタックにおいて、自動的にコンテキストを持ち回ってくれる機能を提供します。非同期処理に対応しているということは、複数のリクエストが同時に処理される場合にもコンテキストが混ざらないようにできるということです。

`AsyncLocalStorage`の基本的なAPIは2つです。それは`AsyncLocalStorage#getStore()`と`AsyncLocalStorage#run()`です。`AsyncLocalStorage#getStore()`は、現在のコンテキストを取得します。`AsyncLocalStorage#run()`は、新しいコンテキストを作成して、そのコンテキストの中で関数を実行します。`AsyncLocalStorage#run()`の第1引数には、新しいコンテキストの値を指定します。第2引数には、新しいコンテキストの中で実行する関数を指定します。

これを使って、先程の例を書き換えると次のようになります。

```js
const metadataStorage = new AsyncLocalStorage();
const getCurrentMetadata = () => metadataStorage.getStore() ?? {};

async function handleRequest(request) {
  await metadataStorage.run({
    ...getCurrentMetadata(),
    requestID: request.id,
  }, async () => {
    const userId = await getUserId(request);
    if (userId === null) {
      log("User not found", getCurrentMetadata());
      return;
    }
    await metadataStorage.run({ ...getCurrentMetadata(), userId }, async () => {
      log("Processing request", getCurrentMetadata());
      await doSomething();
      await doSomethingElse();
      await doSomethingElseAgain();
      log("Finished processing request", getCurrentMetadata());
    });
  });
}
```

ポイントは、`metadataStorage`をシングルトンとして扱えることです。常にシングルトンオブジェクトにアクセスしつつ、`getStore`を呼び出せば現在の状況に応じたメタデータが返ってくるという魔法のようなAPIです。実装は詳しく知らないのですが、JavaScriptエンジン（V8など）レベルの処理で非同期処理を追跡しているはずです。

メタデータを更新したい場合は、例のように`run`を呼び出します。`run`に渡したコールバック関数の中では、`getStore`の結果が更新された状態になります。

## `AsyncLocalStorage`の問題点

`AsyncLocalStorage`は、このように特にサーバーの実装においてとても便利なAPIであり、筆者もよく使っています。

しかし、上の例のように`AsyncLocalStorage`を使う場合には、まだ問題点があります。それが、筆者が**非同期コールバック地獄**と呼んでいるものです。この話はSoftware Design 9月号の非同期処理特集でも少し触れたので、ぜひそちらも読んでみてください（宣伝）。

https://gihyo.jp/magazine/SD/archive/2023/202309

非同期コールバック地獄というのは要するに、`run`を呼び出すたびにコールバック関数がネストしていくことです。正直読みにくいです。メタデータをまめに更新すればするほどコードが読みにくくなっていくというのは、ログの精緻化に対する負のフィードバックとなってしまうためとても良くありません。もともとコールバック地獄というのは昔の非同期処理のやり方を指す言葉ですが、async/awaitによってそれが解決したと思ったら今度は`AsyncLocalStorage`が新しいコールバック地獄をもたらしてしまったわけですね。

ここからが本題です。筆者は、この非同期コールバック地獄を回避し、`AsyncLocalStorage`を使った構造化ロギングを快適に行うためにはどのようなインターフェースが良いか考察しました。その結果として生まれたのが`async-object-stack`です。


:::message
一応Node.jsの`AsyncLocalStorage`には`enterWith`というAPIもあり、これを使えばネストせずに済ますこともできます。しかし、`enterWith`はWinter CGが定義するサブセットに含まれていなかったり、先ほど紹介したAsync Contextプロポーザルにも含まれていなかったりして将来性に欠けます。そのため、今回は考慮に入れません。
:::

## `async-object-stack`のAPI

`async-object-stack`は本質的には`AsyncLocalStorage`のラッパーであり、コンテキストの中身はオブジェクトの配列です。これまでコンテキストに追加されてきたメタデータがスタックのように積み重なっており、取り出すときはそれを全部マージしたものが返ってきます。

`async-object-stack`はある程度ミューテーションも取り入れたAPIになっています。コンテキストを完全にイミュータブルに取り扱うAPIにすると、非同期コールバック地獄が避けられません。

`async-object-stack`を使って先程の例を書き換えると次のようになります。

```js
import { createAsyncObjectStack } from 'async-object-stack';

const stack = createAsyncObjectStack();

async function handleRequest(request) {
  using guard = stack.push({ requestID: request.id });

  const userId = await getUserId(request);
  if (userId === null) {
    log("User not found", stack.render());
    return;
  }

  using guard2 = stack.push({ userId });

  log("Processing request", stack.render());
  await doSomething();
  await doSomethingElse();
  await doSomethingElseAgain();
  log("Finished processing request", stack.render());
}
```

まず`createAsyncObjectStack`を呼び出して、シングルトンの`stack`（`AsyncObjectStack`のインスタンスです）を作っています。

`async-object-stack`の主なAPIは、`push`と`render`（そして後述の`region`）です。

`push`は現在のメタデータに新しいデータを追加します。裏でミューテーションを行うため、コールバックを使わなくてもメタデータが更新されます。

`render`は現在のメタデータを1つのオブジェクトとして返します。この2つでおおよそやりたいことが実現しています。

よく見ると、`push`の戻り値を`using`変数に入れています。`using`構文自体の説明は他の記事に譲りますが、これにより`stack.push`の影響をそのスコープに限定することができます。例えば、`handleRequest`の最初で`stack.push`を呼び出していますが、ここで追加された`requestID`というメタデータは、この関数から出たら消えます。

```js
// ...
await handleRequest(request);
// ここではrequestIDは消えている
log("The end", stack.render());
```

コールバックを避けてミューテーションの方式を採用したということは、生の`AsyncLocalStorage`の中身としては呼び出し元も呼び出し先も同じコンテキストを共有してそこにミューテーションを行うということになります。ミューテーションを行う場合はきちんと後始末をしないと影響が関数の外に漏れてしまうことになりますが、`using`構文を使うことでその問題を解決できます。

:::message
`using`変数に`push`の結果を入れ忘れたら`push`の影響が関数の外に漏れてしまうので、そこは注意する必要があります。将来的には、ある程度TypeScript-ESLintが面倒を見てくれると思います。
:::

ミューテーションを取り入れたインターフェースは、`AsyncLocalStorage`の利点と、少し上で述べた「非同期処理でなければグローバル変数でも大丈夫」という考え方の良いとこ取りになっています。

async/awaitの利点は主に、非同期処理を同期処理のように書けることにあります。`AsyncLocalStorage`はちょうどその一連の処理をスコープとする擬似的なグローバル変数を提供してくれます。ならば後はそれを使って書きやすさに振ったAPIを作れば良いというのが`async-object-stack`の考え方です。

## `region`のルール

ところで、`async-object-stack`のAPIにはもう1つ`region`というものがありました。`async-object-stack`を使うためにはこの`region`を正しく使う必要があります。

先ほど`AsyncLocalStorage`は擬似的なグローバル変数を提供してくれるとは言いましたが、何もしなければプログラム全体で共有された本当のグローバル変数になってしまいます。分けるべきところでは適切にコンテキストを分ける必要があります。

詰まるところ、`region`は`AsyncLocalStorage#run`をラップしただけのものです。コンテキストを共有してほしくないところでは`region`を使ってコンテキストを分ける必要があります。

例えばHTTPサーバーを実装する場合は、リクエスト1件1けんの処理を`region`で囲みます。こうすることで、異なるリクエストに対する処理においてメタデータが混ざることはありません。以下はNode.jsでHTTPサーバーを建てるサンプルからの抜粋です。

```js
const httpServer = createServer((req, res) => {
  logger.region(() =>
    processRequest(req, res).catch((error) => {
      using guard = logger.metadata({ error });
      logger.log("error processing request");
    }),
  );
});
```

このように`processRequest`の呼び出しを`region`で囲むことで、並行して複数の`processRequest`が呼び出されてもメタデータが混ざりません。

この場合以外にも`region`を呼び出すべき場面があります。それは、`Promise.all`などを使って複数の処理に分岐させる場合です。例えば、先ほどの例のここの部分を並行処理に変えたいとします。

```js
  log("Processing request", stack.render());
  await doSomething();
  await doSomethingElse();
  await doSomethingElseAgain();
  log("Finished processing request", stack.render());
```

その場合は、次のように書くべきです。

```js
  log("Processing request", stack.render());
  await Promise.all([
    stack.region(() => doSomething()),
    stack.region(() => doSomethingElse()),
    stack.region(() => doSomethingElseAgain()),
  ]);
  log("Finished processing request", stack.render());
```

こうしないと、`doSomething`と`doSomethingElse`と`doSomethingElseAgain`が同じコンテキストを操作するため、メタデータが混ざってしまいます。

裏の`AsyncLocalStorage`の仕組みを知っていれば`region`をいつ使うべきかはすんなりと理解できるはずですが、`async-object-stack`を使う人が全てそうとは限りません。しかしご安心ください。`region`をいつ使うべきかの原則は比較的単純です。

`region`は、**Promiseをすぐに`await`しない場合に使う**と考えれば基本的にはOKです。次のHTTPサーバーの例（再掲）では、`processRequest()`の返り値がPromiseなので、Promiseを作っています。そして、そもそも外側がasync関数ではないため`await`はできません。よって、Promiseをすぐawaitしない場合に該当するため`region`で囲みます。

```js
const httpServer = createServer((req, res) => {
  logger.region(() =>
    processRequest(req, res).catch((error) => {
      using guard = logger.metadata({ error });
      logger.log("error processing request");
    }),
  );
});
```

次の例（`Promise.all`の例の再掲）は、`doSomething`などは多分async関数なのでPromiseを返します。しかし、それを直接`await`するのではなく`Promise.all`に渡しています。よって、Promiseをすぐ`await`しない場合に該当するため`region`で囲みます。

```js
  await Promise.all([
    stack.region(() => doSomething()),
    stack.region(() => doSomethingElse()),
    stack.region(() => doSomethingElseAgain()),
  ]);
```

この原則ならば、少なくともPromiseの知識がちゃんとしていれば迷うことはないはずです。

別の、もう少し抽象的な考え方も紹介します。Promiseを作ったのにすぐに`await`しない場合というのは、async/awaitによる擬似的な同期処理をするのではなく、新たな並行性を生み出すことに相当しています。作られたPromise（より正確には、Promiseを作ったasync関数）の内部の処理が進行するとともに、それを作った側の処理も並行して進んでいることになるため、ここで並行数が増えていますね。このように複数の処理を並行して走らせる場合はコンテキストを分離する必要がありますから、`region`を使います。

## `Logger`を作る

上の例では`async-object-stack`を生で使っていました。工夫次第で、もう一段抽象化することもできます。おすすめは、ロギングを担当する`Logger`クラスを作り、それをシングルトンにすることです。上の例でいう`stack`を`Logger`のインスタンスに内包するイメージです。

例としては、次のような感じで実装できます。`push`と`region`はほとんどそのまま露出して、`render`を直接呼ぶ代わりに、ログの出力まで担当する`log`メソッドを追加しています。

```ts
export class Logger {
  #stack: AsyncObjectStack;

  constructor() {
    this.#stack = createAsyncObjectStack();
  }

  /**
   * Runs the given function in a separate region.
   */
  region<R>(fn: () => R): R {
    return this.#stack.region(fn);
  }

  /**
   * Add metadata to the current execution.
   */
  metadata(metadata: object): StackGuard {
    return this.#stack.push(metadata);
  }

  /**
   * Emit a log message.
   */
  log(message: string): void {
    const metadata = this.#stack.render();
    console.log(JSON.stringify({ message, ...metadata }));
  }
}
```

この例では標準出力にログを出していますが、お好きな出力先への変更も簡単です。この`Logger`の使用例はGitHubを参照してください。

https://github.com/uhyo/async-object-stack/tree/main/examples/logger

このように`AsyncObjectStack`インスタンスを隠すことは、メタデータの追加の仕方を統一できるという利点があります。自分で`render()`を呼び出してロガーに渡すインターフェースだと、`render()`後にちょっとメタデータをそのまで追加するようなやり方が可能になり、複数の異なるやり方が生まれてしまいます。

```js
// stack.pushを使ってほしいのに……
const metadata = stack.render();
metadata.foo = "bar";
logger.log("message", metadata);
```

上のような`Logger`インスタンスであれば、`metadata`を呼ばないとメタデータを追加できないので、やり方を統一できます。


## 今後

以上のアイデアを形にした`async-object-stack`は、とりあえず必要そうな機能が揃ったためバージョン1.0.0としてリリースされています。

今後やってみたいことは、ロギングに使うとなればパフォーマンスは無視できないため、できる限りパフォーマンスを突き詰めてみたいと考えています。現状はあまりパフォーマンスのことを考えない実装になっています。

## まとめ

この記事では、`AsyncLocalStorage`を活かしつつ快適なインターフェースで構造化ロギングを行うためのAPIを考察しました。

要点としては、まずある程度ミューテーションを取り入れることで非同期コールバック地獄を回避しつつ、ミューテーションの難点を`using`構文で解決するというものです。

また、それでもコンテキストを分ける必要がある場合は`region`を使う必要があります。ここは今回説明したインターフェースにおける難点ではありますが、`region`の使用ルールを比較的明瞭にすることで難しさを緩和しました。

`async-object-stack`は、このような考察の結果として生まれたライブラリです。ぜひ使ってみてください。

https://github.com/uhyo/async-object-stack