---
title: "クライアント側のレンダリング"
---

この章の開始時の内容に対応するコードはリポジトリの`step3`ブランチにあります。

前章では、サーバー側でReactアプリケーションをレンダリングしてRSCプロトコルの文字列に変換する処理までを行いました。その文字列の中で、クライアントコンポーネントはlazy referenceという形で残されていました。

ということで、クライアントサイドではRSCプロトコルからDOMを構成し、さらにクライアントコンポーネントへのlazy referenceを解決してそのレンダリングを行うことになります。

## サーバーサイドの修正

しかしその前に、前章のサーバー側のコードを多少修正する必要があります。というのも、クライアントサイドのコードは（`react-server-dom-webpack`という名前だけあって）webpackと癒着しています。そのため、サーバーから送られてくるJSONをそれに合わせて修正する必要があります。具体的には、次のようにします。

```tsx:src/index.tsx
import rsdws from "react-server-dom-webpack/server";
const { renderToPipeableStream } = rsdws;

import { App } from "./app/App.js";

const bundlerConfig = {
  "src/app/Clock.tsx": {
    Clock: {
      id: "Clock.tsx",
      name: "Clock",
      chunks: ["pika", "chu"],
    },
  },
};

renderToPipeableStream(<App />, bundlerConfig).pipe(process.stdout);
```

具体的には、`bundlerConfig`の中身が`id`, `name`, `chunks`を持つオブジェクトになりました。これらはwebpackのモジュールID、そのモジュールからexportされている名前、そしてそのモジュールをrequireするのに必要なチャンクのIDの一覧を意味します。

以上の修正をしてサーバーを実行すると次の出力が得られます。これを利用してクライアント側を実装していきましょう。

```
M1:{"id":"Clock.tsx","name":"Clock","chunks":["pika","chu"]}
J0:["$","div",null,{"children":[["$","h1",null,{"children":"React Server Components example"}],[["$","p",null,{"children":"Hello, world!"}],["$","@1",null,{}]]]}]
```

## クライアントの実装

とりあえず、動作するクライアントサイドの実装を一気にお見せします。サンプルリポジトリではViteをセットアップしてコードを動かしています（`step4`ブランチを参照してください）。

```tsx:src/client.tsx
// @ts-expect-error
import React, { use } from "react";
import ReactDOM from "react-dom/client";
// @ts-expect-error
import { createFromFetch } from "react-server-dom-webpack/client";
import { Clock } from "./app/Clock.js";

// @ts-expect-error
globalThis.__webpack_chunk_load__ = async (chunkId: string) => {
  console.log(`Chunk '${chunkId}' is loaded`);
};

// @ts-expect-error
globalThis.__webpack_require__ = (moduleId: string) => {
  if (moduleId === "Clock.tsx") {
    return {
      Clock,
    };
  } else {
    throw new Error(`Unknown module ID '${moduleId}'`);
  }
};


const dataFromServer = `
M1:{"id":"Clock.tsx","name":"Clock","chunks":["pika","chu"]}
J0:["$","div",null,{"children":[["$","h1",null,{"children":"React Server Components example"}],[["$","p",null,{"children":"Hello, world!"}],["$","@1",null,{}]]]}]
`;

const root = document.getElementById("app")!;

const chunk = createFromFetch(
  fetch(`data:text/plain;base64,${btoa(dataFromServer)}`)
);

const Container: React.FC = () => {
  return use(chunk);
};

ReactDOM.createRoot(root).render(<Container />);
```

まず、前半のwebpack関連の部分を読み飛ばして後半部分に注目しましょう。今回は`createFromFetch`を`react-server-dom-webpack`からインポートして使用しており、これがRSCのレンダリングにあたってキーとなります。`Fetch`というのはHTTPリクエストを発行できるFetch APIのことを指しています。実際のシチュエーションではサーバー側のレンダリング結果はサーバーから取得することになりますから、このようなデザインは妥当ですね。ちなみに、`react-server-dom-webpack`から提供されているAPIは`createFromFetch`, `createFromReadableStream`, `createFromXHR`の3種です。今回は実際にリクエストを発行しませんが、一番楽に書けるので`createFromFetch`を採用しています。

この`createFromFetch`というのは`fetch`の結果のPromiseを渡すことで`Chunk`というオブジェクトを返します。このオブジェクトは（実際には）サーバーからRSCプロトコルでストリーミングされてくるデータをいい感じに処理してレンダリングすることを担当してくれます。今回は面倒なのでサーバーからのデータはハードコードしてしまっています。

実はこのChunkオブジェクトはPromiseを継承しています。RSC時代のReactでは、Promiseの中身を取り出したい場合は`use`を使うのでしたね。ということで、上のコードでは`createFromFetch`の結果を`use`として表示するだけの`Container`コンポーネントを作ってレンダリングしています。これでRSCプロトコルで送られてきたツリーをレンダリングすることができます。

## モジュールの読み込み

ところで、サーバーからのデータには`M1:{"id":"Clock.tsx","name":"Clock","chunks":["pika","chu"]}`という行があり、これを解決しないと`Clock`コンポーネントをレンダリングできないはずです。実は、`react-server-dom-webpack`では内部でwebpackのAPIを呼び出してこれの解決を試みます。今回はwebpackを使用していないので、ここの部分を補ってあげる必要があります。それが次のコードです。

```ts
// @ts-expect-error
globalThis.__webpack_chunk_load__ = async (chunkId: string) => {
  console.log(`Chunk '${chunkId}' is loaded`);
};

// @ts-expect-error
globalThis.__webpack_require__ = (moduleId: string) => {
  if (moduleId === "Clock.tsx") {
    return {
      Clock,
    };
  } else {
    throw new Error(`Unknown module ID '${moduleId}'`);
  }
};
```

webpackでは、モジュールを読み込むためには先にそのために必要なチャンクを読み込みます。それが`__webpack_chunk_load__`です。そして、読み込みが終わると`__webpack_require__`でその中身を取得することができます。今回は簡単のために`__webpack_chunk_load__`はダミーにしました。そして、`__webpack_require__`も`moduleId`を見てその場で結果を組み立てる感じにします。今回は、`Clock.tsx`というIDのモジュールが読み込まれたら`Clock`をエクスポートするオブジェクトを返します。これで`react-server-dom-webpack`から呼ばれるwebpackのAPIを実装できました。

ということで、これを実行すると無事にレンダリングが成功するはずです。

![実行結果のスクリーンショット](/images/rsc-without-nextjs/screenshot1.png)

実行結果では、「React Server Components example」という見出しと「Hello, world!」というテキストがサーバー側でのレンダリング結果であり、最後の時計がクライアント側でのレンダリングとなっています。ポイントは、クライアント側が持っているのは`Clock`（と`Container`）の実装のみであり、`App`や`Page`などのコンポーネントの実装はクライアント側のコードに含まれていないということです。これがRSCで実現したかったことですね。

おめでとうございます！　以上でRSCの基本的な動作を自分で再現することができました。今回`react-server-dom-webpack`パッケージを使ったのは、今のところ一番ハイレベルなAPIが提供されているように見受けられるからです。Reactのリポジトリには`react-client`や`react-server`というより汎用性の高そうなパッケージも用意されていますが、これらは執筆時点ではnpmに公開されておらず、提供されているAPIもよりローレベルなので採用しませんでした。

なお、今回実装できたのは「SSRしないServer Components」です。今回サーバーからのレスポンスはハードコードしたものの、実際には`Container`がレンダリングされた時点でサーバーにリクエストが飛ぶことが想定されます。つまり、クライアントが読み込まれる前にサーバーが何か準備しているわけではないので、これはSSRではありません。よく「SSRとServer Componentsは別々の概念」と言われますが、そのことが実感できたのではないでしょうか。

## 次の目標

さて、「初めに」では「筆者はRSCのひとつの側面として『（従来の）Reactアプリとうまく統合された便利なテンプレートエンジン』というものがあると考えています。」と述べました。しかし、今のところサーバーとクライアントを別々に実装しており、統合されてる感じがありません。そこで、次はサーバーとクライアントの統合を目指します。

なお、現在の最新情報ではクライアント向けのファイルは`"use client";`と書くというルールになっていますが、今回はそれに縛られる必要は特にないのでガン無視して独自ルールで進みます。ちゃんと`react-server-dom-webpack`をwebpackと併用する場合にはこういったルールが活きてくることになるでしょう。