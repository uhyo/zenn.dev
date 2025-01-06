---
title: "フロントエンドからファイルをダウンロードさせるやり方について"
emoji: "📂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript"]
published: true
---

いまどきのWebアプリにおいては、ファイルのダウンロード機能が必要な場面が多々あります。例えば、バックエンドが生成したCSVデータをファイルとしてダウンロードさせる「CSVダウンロード」機能などです。

:::message
この記事は筆者が趣味で書いたものです。筆者の業務とは一切関係ありません。関係ありませんよ。
:::

今回はAPI[^api]から得られたデータをファイルとしてダウンロードさせたい場合のフロントエンドの実装方法について考察します。

[^api]: この記事では、バックエンドによって実装されHTTPエンドポイントとして公開されているものを指します。これをWeb APIなどと呼ぶ流派もありますが、単にAPIと言っても伝わるので、この記事ではAPIと呼びます。

## 要件

今回考える要件は、前述のとおり、APIから得られたデータをファイルとしてダウンロードさせることです。具体的には、以下のような要件を考えます。

- APIをGETリクエストで呼び出し、そのレスポンスをそのままファイルとしてダウンロードする
- フロントエンドでの何らかのアクション（ボタンクリックなど）によってダウンロードがトリガーされる

追加の要件次第でやり方は変わりますが、とりあえず以上の前提で考えます。

## ベストな方法

とりあえず、筆者が考える一番ベストな方法を紹介します。

それは、**APIのURLにナビゲーションして全部ブラウザに任せる**ことです。

例えば、`/api/csv`がCSVデータを返すAPIのエンドポイントだとします。

フロントエンドでは、これだけでCSVをダウンロードさせられます。

```js
location.href = '/api/csv';
```

ただし、CSVデータがページとして表示されるのではなくファイルとしてダウンロードされるようにするために、API側で[Content-Dispositionヘッダ](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Disposition)を使う必要があるかもしれません。

これで済ませられる場合は、これが一番シンプルだし望ましい方法であるというのが筆者の考えです。

ファイルのダウンロードをブラウザに任せることは多くのメリットがあります。例えば、ダウンロードの中断や再開、キャンセル、エラー時のリトライなどを全部ブラウザのダウンロードUIに任せられるので、フロントエンドで何も実装しなくても機能豊富なダウンロードUXを提供できます。また、ダウンロードを発生させたタブが閉じられたとしても問題なくダウンロードを継続可能です。

**Q.** でもダウンロードの進捗状況を表示したい……

**A.** ブラウザのUIに表示されますよ。

**Q.** でもUXのために自前の進捗表示を用意したい……

**A.** ユーザーが使い慣れたブラウザのUIが使えるということが最高のUXですよ。

ということで、UXのことを考えるならブラウザのファイルダウンロード機能に任せるのがベストでしょう。

しかし、これだけだと記事の内容が薄いので、この方法が何らかの事情で使えない場合のことも考えます。例えば、認証の方式の都合上で単なるナビゲーションができない場合などです。

## よくあるけど微妙な方法

APIのURLにナビゲーションさせる方法を使えない場合に行われがちな方法としては、fetch（またはXHR）でAPIからデータをダウンロードし、それをフロントエンドのバッファに保持して、ダウンロード完了したらそれをファイルとしてブラウザに送る方法があります。

ここでは要件上、ダウンロードの進捗状況を表示しなければならないとしましょう。この場合、fetchのダウンロードがストリーミングで行われることが利用できます。

:::message
以降のサンプル実装では、簡単のためエラーハンドリングは省略しています。
:::

```js
const response = await fetch('/api/csv');
// responseが得られた時点ではまだレスポンスヘッダの受信が完了しただけで、本文の受信はこれから

// 受信データを保存するバッファを用意
const resultBuffer = new ArrayBuffer(0, { maxByteLength: 100 * 1024 ** 2 });
const result = new Uint8Array(resultBuffer)
let offset = 0;

// bodyはReadableStreamオブジェクトである
const body = response.body;

// 受信したチャンクごとに処理する
for await (const chunk of body) {
  resultBuffer.resize(offset + chunk.length);
  result.set(chunk, offset);
  offset += chunk.length;

  // 進捗表示
  console.log(`${offset}バイト受信済`);
}

// ダウンロード完了したら、バッファをBlobに変換して、
// URLを発行してブラウザにダウンロードさせる
const blob = new Blob([resultBuffer.transferToFixedLength()], { type: response.headers.get('Content-Type') });
const url = URL.createObjectURL(blob);

const a = document.createElement('a');
a.download = 'data.csv';
a.href = url;
a.click();
```

この例では、fetchの`response.body`がReadableStreamオブジェクトであることを利用して、ダウンロードの進捗状況を表示しています。ReadableStreamオブジェクトはこのようにfor-await-of構文でデータを受信したそばから処理できます。今回は`console.log`でこれまでに受信したバイト数を表示しています。

もしあらかじめファイルサイズが判明している場合は、それを何らかの形で取得しておけばパーセンテージも表示できるでしょう。

:::message
一応レスポンスの`Content-Length`ヘッダを取得することはできますが、これはレスポンスが圧縮されていた場合には当てにならないので注意しましょう。この場合、`Content-Length`の値は圧縮後のサイズを示している一方、上記のコードのようにfetchで取得した受信データは伸長された状態で取得されるため、正しく受信割合を計算できません。

ちなみに、[fetchで圧縮されたデータを圧縮されたまま取得する方法はありません](https://github.com/whatwg/fetch/issues/1524)。
:::

ちなみに、この例では書き込み先のArrayBuffer (`resultBuffer`) を都度リサイズして保存領域を確保しています。これはES2024の新機能です。また、リサイズ可能なArrayBufferからBlobを作ることができないので、`transferToFixedLength`メソッドを使ってリサイズできないArrayBufferに変換しています。こちらもES2024の新機能です。

細部は異なるかもしれませんが、以上のようなやり方でファイルダウンロードを実装した経験がある方も多いのではないでしょうか。

しかし、せっかく解説したものの、筆者が思うに、これは微妙な方法です。

特に、やはりブラウザのダウンロードUIが使えないことが一番のデメリットです。この方法でも一応最終的にはブラウザのダウンロードUIを介してファイルがダウンロードされることにはなりますが、「タブ内で実際のダウンロードが進行し、完了したらダウンロードUIにファイルが表示される」（ダウンロードUI上は一瞬でダウンロードされたように見える）という点で微妙です。

また、ダウンロードの最中、データをメモリ上に保持しなければならない点も良くありません。データ量が多い場合でもデータの全体をメモリ上に乗せる必要があるため、ネイティブなファイルダウンロードに比べてメモリ使用量が悪化する恐れがあります。

## Service Workerを使う荒業

上で紹介した方法では、fetchの結果としてReadableStreamオブジェクトを得ていましたが、それをブラウザにダウンロードさせるためにBlobに変換する必要があるため、ストリーミングを活かせずに一旦全データをメモリ上に保持しなければなりませんでした。

これを改善して、ReadableStreamを直接ブラウザのダウンロードUIに接続させたいですね。そうすれば、クライアント上でReadableStreamを処理しつつ、ブラウザのダウンロードUIもいい感じに動くはずです。

実は、これを実現するためにService Workerが利用できます。しかし、これは荒業とも言える方法です。

すなわち、ブラウザのダウンロードUIというのは、URLからファイルをダウンロードするときに動作します。そして、Service Workerは、ブラウザがリクエストを送信するときに、そのリクエストをフックして自分でレスポンスを返すことができます。これを利用して、Service WorkerがハンドルするURLからファイルをダウンロードさせることが考えられますね。

つまり、手元にReadableStreamがある場合、それを何とかしてService Workerに送ります。Service Workerはダウンロード用のURLからそのデータをオウム返しします。ブラウザをそのURLにナビゲーションさせると、ブラウザのダウンロードUIが動作して、ダウンロードが始まります。

この仕組みであれば、クライアントで作成されたデータをダウンロードUIに接続させてダウンロードすることもできます。

筆者は記事のためとはいえこれのサンプル用意するの大変だなあと思っていたのですが、探してみたところまさにこれをやってくれるライブラリがありました。ここに書かれていることを試したい場合はこのライブラリのサンプルを見てみてください。

https://github.com/jimmywarting/StreamSaver.js

## File System APIを使う方法

ところで、上記のライブラリのREADMEを見に行くと、今どきは[File System API](https://fs.spec.whatwg.org/)（およびその拡張である[File System Access API](https://wicg.github.io/file-system-access/)）があるからこのライブラリは必要なくなっていくだろうと書かれています。ということで、File System APIを使う方法を見てみましょう。

:::message
ここで取り扱うFile System Access APIについては、Google Chromeに実装されているものの、[Firefox](https://github.com/mozilla/standards-positions/issues/154)および[Safari](https://github.com/WebKit/standards-positions/issues/28)からは反対されています。そのため、一応紹介しますが、このままではあまり将来性が無さそうな仕様であることに注意してください。

また、File System APIを使う以降のサンプルはChromeでしか動作しません。
:::

具体的な実装はこのようになります。

```js
// ユーザーに保存先を選択してもらう
const handle = await showSaveFilePicker({
  suggestedName: "data.csv",
  types: [
    {
      description: "CSVファイル",
      accept: {
        "text/csv": [".csv"],
      },
    },
  ],
});

const response = await fetch('/');
const body = response.body;
let offset = 0;

const writable = await handle.createWritable();

for await (const chunk of body) {
  // ファイルに書き込む
  await writable.write(chunk);
  
  offset += chunk.length;

  // 進捗表示
  console.log(`${offset}バイト受信済`);
}

// ファイルを閉じる
await writable.close();
```

要するに、File System Access APIに由来する`showSaveFilePicker`という関数を使ってユーザーに保存先を選択してもらいます。

そうするとFileSystemFileHandleオブジェクトが得られるので、それに対して書き込みを行うことで、ユーザーが選択したファイルへのダウンロードができます。

この方法ではユーザーの手元にファイルを保存するという目的は達成できるものの、ブラウザのダウンロードUIには何も表示されません。そもそもダウンロードではないからですね。

そのため、前述のUXという観点では結局おすすめできません。File System Access APIのissueには[ダウンロードUIと接続したいという提案](https://github.com/WICG/file-system-access/issues/379)があります（issueを建てたのは前述のライブラリを作成した人です）が、そこまで興味を持たれていないようです。

## まとめ

今回は、APIから得られたデータをファイルとしてダウンロードさせる方法について考察しました。

やはり、ブラウザのダウンロードUIを使うのが一番シンプルで望ましい方法であるというのが筆者の考えです。しかし、そのためにはダウンロード用のURLにナビゲーションできるようにAPIを作る必要があります。サーバーサイドも交えて、これができるように設計するのが望ましいでしょう。

どうしてもそれができない場合には、最善のUXを諦めるか、あるいは（ライブラリがあるとはいえ）Service Workerを持ち出す大がかりな方法を使う必要があります。

ファイルシステムというのはどうしても高いセキュリティが求められる領域ですから、自由なアクセスには制限がかかります。Web標準の発展という観点から見てもなるべくブラウザに任せるのが良さそうです。