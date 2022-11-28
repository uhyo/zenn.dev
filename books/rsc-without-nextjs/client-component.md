---
title: "Client Componentを分割しよう"
---

この章の開始時の内容に対応するコードはリポジトリの`step2`ブランチにあります。

前章では、nodeに`--conditions react-server`というオプションを渡していたことからも分かるように、Server Components用の環境でレンダリングしていました。そのため、`useEffect`や`useState`などは使用できませんでした。

次はサーバー用コンポーネントとクライアント用コンポーネントを混在させることを目指します。

## Server Componentsから見たクライアント用コンポーネント

前章でも説明したように、RSCは1つのアプリケーションをサーバーとクライアントで分担してレンダリングする仕組みです。そうなると、サーバー側コンポーネントの環境ではクライアント用コンポーネントはレンダリングせずにそのまま残すのが妥当です。

具体的には、サーバー側の環境から見たクライアント用コンポーネントは具体的な実装（関数コンポーネントやクラスコンポーネント）ではなく、「これはクライアント用コンポーネントだよ」ということを表す特殊な値であると考えられます。

この特殊な値が何であるかは、`react-server-dom-webpack`のコードを読むと判明します。時間に余裕のある方はコードを読んで探してみましょう。ここでは忙しい方のために、答えを用意してあります。`Clock`コンポーネントを次のように定義してみましょう。

```tsx:src/app/App.tsx
// import { Clock } from "./Clock.js";
import { Page } from "./Page.js";

const Clock = {
  $$typeof: Symbol.for("react.module.reference"),
  filepath: "src/app/Clock.tsx",
  name: "Clock",
} as unknown as React.ComponentType;

export const App: React.FC = () => {
  return (
    <Page>
      <p>Hello, world!</p>
      <Clock />
    </Page>
  );
};
```

つまり、サーバー側から見たクライアントコンポーネントというのは、`$$typeof`プロパティに`Symbol.for("react.module.reference")`を持つオブジェクトとして表現されます。さらに、このオブジェクトには`filepath`と`name`が必要です。意味としては、`src/app/Clock.tsx`というモジュールから`Clock`という名前でエクスポートされているオブジェクトであるという意味になるでしょう。

## クライアント用コンポーネントをレンダリングする

上のコードをレンダリングするにはもう一つ準備が必要です。`src/index.tsx`（`renderToPipeableStream`を呼び出しているところ）を次のように修正しましょう。

```tsx
// @ts-expect-error
import rsdws from "react-server-dom-webpack/server";
const { renderToPipeableStream } = rsdws;

import { App } from "./app/App.js";

const bundlerConfig = {
  "src/app/Clock.tsx": {
    Clock: {
      pika: "chu",
    },
  },
};

renderToPipeableStream(<App />, bundlerConfig).pipe(process.stdout);
```

このように、`renderToPipeableStream`の第2引数に`bundlerConfig`を渡すようにしました。この`bundlerConfig`というのは、先ほどの`filepath`と`name`をキーとした辞書オブジェクトだと考えましょう。

以上の修正を行い、`src/index.tsx`を実行してみましょう。すると、次のような結果になるはずです。

```
M1:{"pika":"chu"}
J0:["$","div",null,{"children":[["$","h1",null,{"children":"React Server Components example"}],[["$","p",null,{"children":"Hello, world!"}],["$","@1",null,{}]]]}]
```

従来の`J0`という行の上に`M1`という行が加わりました。察するに、これは「モジュール読み込み」命令を表すものです。`filepath: "src/app/Clock.tsx"`と`name: "Clock"`に対応するモジュールを読み込むという意味です。この行に含まれるJSONは適当なので今回は意味がありませんが、実際にはこの情報を頼りにクライアント側でモジュール読み込みを行うことになるでしょう。

次の`J0`の行を見ると、クライアントコンポーネント`Clock`に対応するのは`["$","@1",null,{}]`という部分です。つまり、クライアントコンポーネントが`@1`という文字列で表されています。`1`というのは`M1`に対応しており、`M1`の行で読み込まれたものをここで参照するという意味であることが分かります。ちなみに、このプロトコルでは`@`はlazy referenceを意味するそうです（lazyではないreferenceは`$`）。ここでは、非同期読み込みに対応しているという意味だと思いましょう。クライアントサイドでまだ当該モジュールが読み込まれていない場合はサスペンドされることが想像できますね。

以上で、最もベーシックなコンポーネントの分割は完了です。サーバーサイドではこのようにServer Componentsのプロトコルに沿った文字列が出力され、それをクライアント側でレンダリングすることで1つのアプリケーションのレンダリングが完了します。

ということで、次の章ではクライアントサイドを実装してみましょう。

現時点のコードが`step3`ブランチにあります。