---
title: "Suspenseの挙動を観察しよう"
---

前章で、とりあえずコンポーネントをサスペンドさせることができました。しかしあれだけではまだ使い物になりませんね。実用できるものを作るためにはまだ知識が足りていません。

そこで、次はSuspenseの挙動の観察を通してSuspenseの理解を深めましょう。

# コンポーネントの再レンダリングの挙動を観察しよう

前回作った`AlwaysSuspend`コンポーネントはPromiseをthrowしますが、そのPromiseは1秒後に解決されるのでした。しかし、画面の表示を観察しても、表示はずっと「Loading...」のままです。では、この1秒というのは何の意味があったのでしょうか。その謎は、`AlwaysSuspend`に`console.log`を仕込んでみると分かります。

```diff tsx
 const AlwaysSuspend: React.VFC = () => {
+  console.log("AlwaysSuspended is rendered");
   throw sleep(1000);
 };
```

このようにすると、1秒ごとに「AlwaysSuspended is rendered」とコンソールに表示され続けることが確認できます。これが意味することは、「`AlwaysRendering`の再レンダリングが1秒ごとに試みられている」ということです。これはなぜなのでしょうか。

その理由は、throwされたPromiseは**サスペンドがいつ終了すると見込まれるか**を示すものだからです。普通のコンポーネントは無限にローディングを続けず、いつかローディングが完了するものです。Promiseが解決されることで、ローディングの終了が表されるというのが意図されている実装です。

そして、ローディングが終了したらそれに合わせて画面を書き換えなければいけません。つまり、`fallback`の内容を片付けてサスペンドしたコンポーネントの本来の内容を表示するという作業が必要なはずです。Reactは、これを**サスペンドしたコンポーネントの再レンダリング**という形で行います。普通、サスペンドしたコンポーネントから投げられたPromiseが成功裏に解決された場合、コンポーネントのレンダリングを再度試みれば、今度はサスペンドしないでレンダリングが成功することが期待されます。それにより、再レンダリングすればローディング完了後の画面になるというわけです。

ところが、この`AlwaysRendering`は再レンダリングしたらまた新たなPromiseをthrowします。つまりこれは**二度寝**ですね。`AlwaysRendering`は「あと1秒寝かせて……」を無限に繰り返すコンポーネントだったのです。これによりずっとサスペンド状態となり、だから画面がずっと「Loading...」のままだったのです。

# サスペンドを終了させる

しかし、これだけだと面白くないので、サスペンドが終わったときの挙動も見てみたいですね。いきなりちゃんとしたものを作り込むのは結構大変なので、またもや変なコンポーネントを導入します。その名も`SometimesSuspend`です。これは50%の確率でサスペンドし、それ以外の場合はレンダリングに成功します。

```tsx
export const SometimesSuspend: React.VFC = () => {
  if (Math.random() < 0.5) {
    throw sleep(1000);
  }
  return <p>Hello, world!</p>;
};
```

これを`AlwaysSuspend`の代わりに使ってみましょう。

```diff tsx
 <div className="text-center">
   <h1 className="text-2xl">React App!</h1>
   <Suspense fallback={<p>Loading...</p>}>
-    <AlwaysSuspend />
+    <SometimesSuspend />
   </Suspense>
 </div>
```

何回か画面を更新してみると、「1秒間Loading...と表示された後に『Hello, world!』と表示される」や「いきなり『Hello, world!』と表示される」などといった挙動が観察されるはずです。1秒間Loading...と表示された場合は、1回目の`SometimesSuspend`レンダリングではPromiseがthrowされたのでサスペンドされ、1秒後に行われた2回目のレンダリングではPromiseをthrowしなかったことが伺われます。

**練習問題:** このアプリケーションにおいてLoading...と表示される秒数の期待値を求めよ．ただし`sleep(1000)`以外の要因による時間経過は無視してよい．

:::details 解答
  0 + 1⁄2 + 1⁄4 + … = 1 \[秒\]
:::

なお、一度「Hello, world!」と表示されたあとに勝手に再度Loading...の状態に戻ることはないことに注意してください。これは、一度レンダリングが完了すれば、もう`SometimesSuspend`が再レンダリングされることはないからです。

逆に言えば、`SometimesSuspend`を何らかの理由で再レンダリングさせれば、再びサスペンドする可能性があります。具体的には、例えば`App`に状態を持たせれば状態の更新時に再レンダリングが起こるでしょう。

```diff tsx
 function App() {
+  const [count, setCount] = useState(0);
   return (
     <div className="text-center">
       <h1 className="text-2xl">React App!</h1>
       <Suspense fallback={<p>Loading...</p>}>
         <SometimesSuspend />
+          <button className="border p-1" onClick={() => setCount((c) => c + 1)}>
+            {count}
+          </button>
       </Suspense>
     </div>
   );
 }
```

このようにすれば、ボタンを押して`count`を更新する（＝`App`を再レンダリングする）たびに、`SometimesSuspend`がサスペンドしたりしなかったりして、その結果としてLoading...に戻ったり戻らなかったりするのが観察できます。

# サスペンド終了時はどこまでが再レンダリングされるのか？

するどい読者の方は、まだ疑問があるのではないでしょうか。それは、サスペンドが終了した際に（より正確には投げられたPromiseが解決した際に）再レンダリングされるというのは、具体的にはどのコンポーネントが再レンダリングされるのかということです。

「そんなのサスペンドしたコンポーネントが再レンダリングされるに決まっているだろ」とお思いかもしれません。それはその通りです。しかし、前章で以下のように書いてあったことを思い出してください。

> これはReactが提供する一貫性保証の一部であり、ある瞬間にレンダリングされたコンポーネントツリーが部分的に表示されてしまうようなことを防ぐためであると思われます。全部表示できるか、全部表示できないかのどちらかなのです。

> ここから、`Suspense`コンポーネントが実は**サスペンドの境界を定義する**役割を持っていることがお分かりになるでしょう。

つまり、サスペンドの境界は`Suspense`コンポーネントであり、レンダリングの一貫性を保つにはサスペンド終了時に`Suspense`の中を全部再レンダリングしなければならないはずです。

この挙動を実際に確かめるために、新たな補助コンポーネントを追加しましょう。

```tsx
type Props = {
  name: string;
};

export const RenderingNotifier: React.VFC<Props> = ({ name }) => {
  console.log(`${name} is rendered`);

  return null;
};
```

これは、画面には何も表示されないが、レンダリングされた（関数コンポーネントが呼び出された）ら`console.log`するだけのコンポーネントです。先ほど`AlwaysSuspend`でやったのと同様に、レンダリングの様子をこれで追ってみましょう。`App`内の2箇所に配置してみます。

```diff tsx
 function App() {
   const [count, setCount] = useState(0);
   return (
     <div className="text-center">
       <h1 className="text-2xl">React App!</h1>
+      <RenderingNotifier name="outside-Suspense" />
       <Suspense fallback={<p>Loading...</p>}>
         <SometimesSuspend />
+        <RenderingNotifier name="inside-Suspense" />
         <div>
           <button className="border p-1" onClick={() => setCount((c) => c + 1)}>
             {count}
           </button>
         </div>
       </Suspense>
     </div>
   );
 }
```

こうすると、例えば以下のようなログが表示されます（`SometimesSuspend`が1回サスペンドした場合）。

```
outside-Suspense is rendered
inside-Suspense is rendered
inside-Suspense is rendered
```

つまり、サスペンド解除時は`Suspense`の中に配置した`RenderingNotifier`は巻き込まれて再レンダリングされた一方、外に配置した`RenderingNotifier`は再レンダリングされませんでした。

結論としては、**サスペンド解除時はサスペンドした`Suspense`の中身が再レンダリングされる**ということになります。再レンダリングされるとはいっても結局DOM更新は最適化されるので神経質にならければいけない場面は多くないでしょうが、頭の片隅に置いておきましょう。

では、そろそろ観察ばかりで飽きてきた頃でしょうから、また実装を進めていきましょう。