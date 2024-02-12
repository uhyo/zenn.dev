---
title: "Node.jsから呼び出したWASMバイナリ（Rust製）と非同期に通信したい話"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nodejs", "rust", "wasm"]
published: true
---

どうもこんにちは。筆者はここ1年くらい**nitrogql**というTypeScript + GraphQL向けコード生成ツールを開発しています。（初手宣伝）

https://nitrogql.vercel.app/

https://github.com/uhyo/nitrogql

このツールの本体はRustで書かれており、コンパイルするとWASMバイナリが生成されます。このWASMバイナリをNode.jsから呼び出すようなラッパーを作って、コマンドラインツールとしてnpmで公開しています。

その性質上、Node.js側とWASM側で通信（データのやり取り）が発生します。特に、設定ファイルなどが`.js`や`.ts`で書かれていても読み込む機能があり、その際はRust側からNode.js側に制御を渡してNode.js側でファイルを読み込み、結果をRust側に返すようになっています。

実は、nitrogqlの（Rust側）コードにはこれまで非同期処理が含まれていませんでした。しかし、パフォーマンスのことなどを考えると非同期処理に真面目に向き合う必要があると感じ、最近リリースしたバージョン1.6.2から取り入れられています。

このようなNode.js + WASM (Rust) の場合において非同期処理の実装がなかなか一筋縄ではいかなかったので、この記事では筆者がうまく非同期処理を動かすまでの過程を紹介します。

:::message
筆者はRustの非同期プログラミングに関してはかなり初心者です。そのため、この記事に書いているないようが正解かどうかも定かではありません。お前よりうまくできるよという方はぜひ知見を共有してください。
:::

## 非同期処理導入前のnitrogqlの仕組み

Node.js (JavaScript) は、基本的にはシングルスレッドで動くものです。そしてWASMも、マルチスレッド対応をWASMの仕様に追加する動きもあるものの、今のところはシングルスレッドで動きます。

そして、Node.jsとWASMの通信は、WASMモジュールからエクスポートされた関数をJavaScript側で呼び出すことで行ないます。

この結果として、Node.jsとWASMは同じスレッドの上で動くのです。DevToolsでコールスタックを取得すると、次の画像のようにJavaScriptとWASMがコールスタックを共有していることが分かります。

![JavaScriptとWASMがコールスタックを共有している様子](/images/nodejs-wasm-async-communication/mixed-stacktrace.png)

JavaScriptからWASMの関数を呼び出すとコールスタックにWASMの関数が積まれることになり、その逆も同様です。この画像では一番上に見える「GenericJSToWasmWrapper」より下がWASMの部分であり、真ん中下にある「execute_node」でJavaScriptに戻ってきています。

このように、Rust側で同期関数として書かれたものはJavaScriptからも同期的な関数として見えます。もちろん、Rust側からJavaScriptの関数を呼び出すときも同様です。Rustから呼び出されるJavaScriptの関数は同期的でなければいけませんでした。

## 非同期処理の必要性

nitrogqlは現状あまり速いとは言えませんが、問題はほとんどがNode.js側にあります。Node.js側で`.js`や`.ts`を実行するところが圧倒的に遅く、ボトルネックとなっています。

この処理（`execute_node`）はRust側から呼び出される部分であるため、同期的に行う必要がありました。当該の`.js`がESMで書かれていても実行したいなどの事情から、この部分の処理は[child_process.execSync](https://nodejs.org/dist/latest-v20.x/docs/api/child_process.html#child_processexecsynccommand-options)を使って実装されていました。別プロセスで`node`を起動して`.js`などを実行するというものです。この方法では、余計なオーバーヘッドがあることは想像に難くありません。

このオーバーヘッドを解消するためには、Node.js側で`.js`を実行するところを非同期処理にする必要がありました。

## 非同期処理実装の方針

ここを非同期にすると、Rust側からJavaScript側の関数を呼び出すところが非同期になるということです。Rust側において、非同期処理の中心にあるのは[`Future`トレイト](https://doc.rust-lang.org/std/future/trait.Future.html)です。これはおおよそJavaScriptの`Promise`に相当するものと考えればよいでしょう。

そのため、理想的にはRust側からJavaScriptを呼び出すとFutureが得られて、それを待つことでJavaScript側の非同期処理が完了するまで待機するという流れになります。

ただ、JavaScriptとWASMのコミュニケーションに関しては、（将来的には[Component Model](https://github.com/webassembly/component-model)によって進展がありそうですが）現状は数値しかやり取りできません。その縛りの中でやる必要があります。

調べると[wasm-bindgen-futures](https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen_futures/)というのが見つかりますが、今回はこれは採用していません。そもそも、別にbindをgenしたくありません。これまでのところRustコンパイラ自体の機能でWASMを出力でき、それを直接Node.jsから呼び出すという構成になっていたため、中間レイヤーを増やさずに済むならそれが望ましいと考えました。

ということで、必要な実装は自前で用意することになりました。目標は、JavaScript側の非同期処理をRust側で`Future`として表すことです。そうすれば、残りは普通の非同期Rustプログラムとして実装できます。

### 非同期処理のためのインターフェース

前述のように、RustとJavaScriptの間のやり取りは数値のみです。「`Promise`オブジェクト」のような高尚なものは使えません。そのため、非同期処理の実務を担当するインターフェースをこの条件下で用意する必要があります。

一応、文字列を受け渡すことはできます。WASMが持っているメモリをJavaScript側から読み書きできるので、Rust側から`alloc`と`free`を提供して、JavaScript側からメモリに文字列を書き込んでポインタを渡せばいいのです。

ということで、方針としては、Rust側では非同期処理のIDを表す数値を発行し、それをJavaScript側に渡すことにしました。JavaScript側では、その数値を受け取って非同期処理を開始し、終了したらその数値をRust側に渡すという流れです。そうなると、JavaScriptの非同期処理を表すRust側の`Future`の実態はそのIDになります。

関連する実装を抜き出して紹介します。

```rust
pub struct TicketId(u32);

pub struct Ticket {
    pub id: TicketId,
}

impl Future for Ticket {
    type Output = Result<String, ()>;

    fn poll(
        self: std::pin::Pin<&mut Self>,
        ctx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Self::Output> {
        TICKETS.with(|tickets| {
            let mut tickets = tickets.borrow_mut();
            if let Some(result) = tickets.take_result(self.id) {
                return Poll::Ready(result);
            }
            let waker = ctx.waker().clone();
            tickets.register_ticket(self.id, waker);
            Poll::Pending
        })
    }
}
```

`poll`の中身の説明は省略しますが、この`Ticket`の結果が得られたことを通知するための`Waker`を取得し、それをスレッドローカル変数として用意されたレジストリに登録しておきます。

### 非同期処理の開始

`TicketId`をJavaScript側に渡すところの処理はこのようになります。

```rust
// Ticketを発行する
let ticket: Ticket = issue_string_ticket();
// IDをJavaScript側に渡して非同期処理開始
// （文字列であるcodeはポインタと長さの組で渡す）
unsafe { execute_node(code.as_ptr(), code.len(), ticket.id.into()) };
// 結果が返ってくるまで待つ
let result = ticket.await;
// ...
```

非同期処理を開始するところの処理がちょっと生々しくて、`Ticket`の生成と非同期処理の開始が一体化していないのが気になりますが、そこを超えれば普通の非同期Rustです。

### 非同期処理の完了

JavaScript側から非同期処理の結果を通知してもらうために、このような関数をWASMモジュールからエクスポートします。

```rust
#[no_mangle]
pub extern "C" fn execute_node_ret(id: u32, is_ok: u32, result: *const u8, result_len: usize) {
    TICKETS.with(|tickets| {
        let mut tickets = tickets.borrow_mut();
        let id = TicketId(id);
        let result = unsafe {
            let slice = std::slice::from_raw_parts(result, result_len);
            String::from_utf8(slice.to_vec()).expect("invalid utf8")
        };
        let ticket = tickets
            .string_tickets
            .get_mut(&id)
            .expect("invalid ticket id");
        ticket.result = if is_ok != 0 {
            Some(Ok(result))
        } else {
            Some(Err(()))
        };
        ticket.waker.wake_by_ref();
    });
    drive();
}
```

ざっくり言えば、JavaScript側からもらったIDを頼りに先ほど保存しておいたWakerを呼び出す処理が書かれています。これにより、Rust側で当該`Future`を`.await`していれば、そこから処理が再開されます。

## 非同期ランタイム

ところで、Rustで非同期処理を動かすには非同期ランタイムというものを導入する必要があります。[tokio](https://tokio.rs/)などが有名ですね。非同期ランタイムは、Node.jsで言うところのイベントループを担当するものだと思えば、ざっくりとしたイメージがつくでしょう。シングルスレッドかマルチスレッドかによっても異なりますが、シングルスレッドならば`Promise`が処理される仕組みとそんなに変わりません。

前述のtokioはWASMにも対応しているとされていますが、今回は採用できませんでした。理由は、tokioの持つイベントループをカスタマイズできなかったからです。tokioは現状、タイマーとIOを非同期のイベントとして対応していますが、独自のイベントをそこに増やすことはできなそうでした。

代わりに、今回採用した非同期ランタイムは[async-executor](https://github.com/smol-rs/async-executor)です。これは非常にシンプルなAPIを持ち、今回の要件に合うような使い方も可能なものでした。

このasync-executorをラップして2つの関数を作りました。`spawn`と`drive`です。

```rust
use async_executor::LocalExecutor;

pub struct Runtime {
    inner: LocalExecutor<'static>,
}

impl Runtime {
    /// Create a new async runtime.
    pub fn new() -> Self {
        Self {
            inner: LocalExecutor::new(),
        }
    }

    /// Spawn a future onto the runtime.
    pub fn spawn<F>(&self, future: F)
    where
        F: std::future::Future<Output = ()> + 'static,
    {
        self.inner.spawn(future).detach();
    }

    /// Drive the runtime until there is no more work to do.
    pub fn drive(&self) {
        loop {
            if !self.inner.try_tick() {
                break;
            }
        }
    }
}
```

`spawn`は非同期ランタイムに`Future`を登録するものです。これにより、`async-executor`が`Future`の結果が出たかどうか、結果が出ていれば後続のタスクを実行するといったことを管理してくれます。

ただし、`spawn`を呼び出すだけで裏で自動的に処理が進むわけではありません。WASM本体はあくまでシングルスレッド・同期実行しか無いため、裏という概念が無いのです。

そのため、イベントループを明示的に回す必要があります。そのために用意された関数が`drive`です。これを呼び出すと、その時点で先に進めることができるタスクがあれば全て進めます。

簡単に言えば、`drive`は手動でイベントループを1回回す操作に相当します。もうタスクが無くなるまでタスクを処理し続けるのは、settle済みのPromiseをすべて処理するのと同じと考えられます。

上記の`execute_node_ret`（JavaScriptから非同期処理の結果を受け取る関数）の最後で、よく見ると`drive`が呼び出されていることが分かります。これは、非同期処理の結果が出たことで新たに再開可能なタスクが発生するため、明示的に`drive`を呼び出すことでそれを処理してもらうためです。

`drive`の呼び出し自体は同期的であることに注意してください。つまり、その時点で待たずに実行できるタスクを全部実行してしまい、それが終わったら関数から返ってきます。JavaScript → `execute_node_ret` → `drive` という経路が意味するのは、これによって「JavaScriptのイベントループが回っており、必要に応じてその一部としてWASM側のイベントループを回す」という仕組みができているということです。回り切ったらJavaScriptのイベントループに制御が戻ります。

![JavaScriptのイベントループからWASMのイベントループが呼び出される様子の模式図](/images/nodejs-wasm-async-communication/event-loop.png)

説明の順番が前後しましたが、このような`drive`の実装に必要なAPIを提供してくれている非同期ランタイムを探した結果、async-executorに行きついたのです。WASMにコンパイルしても普通に動いたのでとても助かりました。

そして、イベントループをどのように回すかというところに非同期ランタイムの個性が出るのだろうということも学びました。先ほど例に出たtokioの場合、現在のところI/Oとtimerという2つのdriverをサポートしているとされています。この用語を借りるならば、今回の実装はJavaScript側からの通知に対応するdriverを実装したと言うことができるのだと思います。

この実装では、`drive()`を呼び出す箇所を増やすことで、WASM側のイベントループが回るきっかけを増やすことができます。現状では、WASMバイナリの実行開始時とJavaScriptからの通知を受けたときの2箇所で`drive()`を呼び出しています。

ちなみに、多くの非同期ランタイムでは便利な`block_on`という関数が提供されていますが、WASMバイナリでは（今のところ）これは使えません。`block_on`は与えられた`Future`が完了するまでブロックする関数です。しかし、WASMにはブロックする（裏でスレッドを休止する）機能がありません。呼び出したら無限ループに陥ってしまいました。そのため、WASMからJavaScript側で制御を返すことで待つ必要がありました。

## まとめ

この記事では、RustからコンパイルされたWASMをJavaScriptから利用している場合において、JavaScript側での非同期処理をRust側で`Future`として扱う方法について紹介しました。

具体的には、既存の非同期ランタイム（async-executor）をラップしてJavaScript側とのやり取りを行なうインターフェースを用意し、JavaScript側での非同期処理完了のタイミングでWASM側のイベントループを回す仕組みを実装しました。

この方法が良い方法なのかは良く分かりませんが、現状の制約の中でミニマルな仕組みを作れたのではないかと思います。筆者は正直なところRustの非同期処理に詳しくないので、もっと良い方法があるよという方はぜひ教えてくださるとありがたいです。

実際の実装についてはこちらを参照してください。

https://github.com/uhyo/nitrogql/pull/45

ちなみに、この実装はnitrogql CLIのパフォーマンスを向上させるために行なっていましたが、実装完了後に試してみると特に速くなっていませんでした。非同期化したことで可能になった最適化もあるはずなので今後に期待しましょう。

🥲