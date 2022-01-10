---
title: "startTransitionを使ってみよう"
---

まずは、これらの機能がどんな機能なのか簡単に解説します。`startTransition`についてはReact 18のalpha版発表時に公開された機能であり、以前に以下の記事でも解説しているので合わせてご覧ください。

https://zenn.dev/uhyo/articles/react-18-alpha-essentials#starttransition

# `startTransition` とは何か

`startTransition`の方が簡単なので、こちらから解説します。`startTransiton`はReact 18以降で追加される予定の機能で、`react`パッケージからインポートして使用する関数です。

```ts
import { startTransition } from "react";
```

そして、`startTransition`はコールバック関数を受け取り、その関数をすぐに呼び出します。コールバック関数内ではステート更新を行いましょう。

```ts
startTransition(() => {
  setSomeState(/* ... */);
});
```

このように`startTransition`の中で行われたステート更新は**トランジション**であると見なされます。トランジションは、端的に言えば優先度の低いステート更新です。

トランジションとしてマークされたステート更新は優先度が低いので、トランジション（としてマークされたステート更新によって引き起こされた再レンダリング）は中止されて一旦後回しにされる可能性があります。前の本で話題に上がったサスペンドもそうでしたが、このようにReact 18ではレンダリングが中断・中止される可能性があるので、レンダリング中に副作用を発生させないように一層注意しなければいけないのでした。

# トランジションを使ってみる

では、さっそくコードを書いていきましょう。ベースとなるコードはこちらのリポジトリのmasterブランチに用意しました。

https://github.com/uhyo/react-suspense-handson

また、`chapter/use-startTransition`ブランチのこの章の内容を反映したコードがあります。

まずは下準備をしていきましょう。前の本の知識を活かして、1秒間サスペンドするコンポーネントを用意します。今回は雑にグローバル変数を使っていきます。

```tsx
let sleeping = true;

const Sleep1s: React.VFC = () => {
  if (sleeping) {
    throw sleep(1000).then(() => {
      sleeping = false;
    });
  }
  return <p>Hello!</p>;
};
```

これを`App`の中でとりあえず使ってみましょう。

```tsx
<Suspense fallback={<p>Loading...</p>}>
  <Sleep1s />
</Suspense>
```

これで画面を更新すると、1秒間「Loading...」と表示されたのち、1秒後に「Hello!」と表示されます。

ここでは、この`Sleep1s`を「表示に1秒かかるコンポーネント」と見なします。よって、次は「トランジションの結果として`Sleep1s`が表示される」というように修正してみます。すると、`App`はこんな感じになるでしょう。

```tsx
function App() {
  const [sleepIsShown, setSleepIsShown] = useState(false);
  return (
    <div className="text-center">
      <h1 className="text-2xl">React App!</h1>
      <Suspense fallback={<p>Loading...</p>}>
        {sleepIsShown ? <Sleep1s /> : null}
      </Suspense>
      <p>
        <button
          className="border p-1"
          onClick={() => {
            startTransition(() => {
              setSleepIsShown(true);
            });
          }}
        >
          Show Sleep1s
        </button>
      </p>
    </div>
  );
}
```

こうすると、確かに「Show Sleep1s」ボタンを押すと1秒後に「Hello!」と表示されます。ただ、すこし異変がありますね。ボタンを押してから1秒間の間（`Sleep1s`がサスペンドしている間）、「Loading...」が表示されないのです。

つまり、ここでは「**トランジションの結果として起きたサスペンドのフォールバック表示をしない**」という効果が起きています。トランジションが実用上重要なのはこの点があるからでしょう。時間がかかるのは避けられないとはいえ、サスペンドによって起こるローディング表示を飛ばしてサスペンド前→表示完了後という画面遷移が可能なのです。これはサスペンド時間が短い（100ミリ秒とか）場合に特に有効で、一瞬ローディングが表示されることによるちらつきを防ぐことができます。

実はこの挙動も「トランジションの優先度が低いこと」で説明可能です。次章で詳しく見ていきましょう。

