---
title: 表の世界のペンディング状態を得る
---

最後に、実は表の世界から「今裏の世界があるかどうか」を取得する方法が用意されています。次はこれを解説します。

# `useTransition`フックを使用する

具体的には、`useTransition`フックを使用します。このフックは2つの値をタプルで返すフックであり、1つは`isPending`フラグ、もう1つは`startTransition`関数です。

`startTransition`関数というのは普通に`react`からインポートできるものと同じ効果を持ち、トランジションを開始することができます。そして、そのトランジションの最中（裏の世界が存在している間）、同じ`useTransition`フックから返される`isPending`フラグが表の世界では`true`になります。

このことを確かめるために、`App`をこんな感じに直してみましょう。

```diff tsx
 function App() {
   const [counter, setCounter] = useState(0);
   const time = useTime();
+  const [isPending, startTransition] = useTransition();
   return (
     <div className="text-center">
       <h1 className="text-2xl">React App!</h1>
+      <p className={"tabular-nums" + (isPending ? " text-blue-700" : "")}>
         🕒 {time}
       </p>
       <Suspense fallback={<p>Loading...</p>}>
         <ShowData dataKey={counter} />
       </Suspense>
       <p>
         <button
           className="border p-1"
           onClick={() => {
             startTransition(() => {
               setCounter((c) => c + 1);
             });
           }}
         >
           Counter is {counter}
         </button>
       </p>
     </div>
   );
 }
```

この例では、後半で使用されている`startTransition`は`react`からインポートしたものではなく、`useTransition`フックから返ってきたものになっている点に注意してください。そして、`isPending`が`true`の間は時計の文字を青くするようにしました。

この例でボタンを押すと、トランジションの最中（`ShowData`が新しい値で更新されるまでの間）は時計の文字が青くなるのが観察できます。

このように、`useTransition`フックを使用することで、今トランジションが実行中かどうかを表の世界から知ることができます。これは、サスペンド中に、`Suspense`の`fallback`に頼らずに自前で「ローディング中……」のような表示をしたい場合に役に立ちます。ちなみに、裏の世界では`isPending`は`false`となります。

前回までの図に`isPending`の状態を書き足してみると、このようになります。

![isPendingの状態を書き足した図](/images/react-concurrent-handson-2/pending-1.png)

表の世界では、`setCounter`の実行直後に`isPending`が`true`になって再レンダリングされます。それと同時に裏の世界が作られます。その後、サスペンドが終了して裏の世界の状態が表の世界に反映されるまで、表の世界では`isPending`は`true`のままです。

ちなみに、counter=0 の状態からボタンを押したときに表の世界で`isPending`が`true`の状態がレンダリングされるのは、裏の世界で counter=1 の状態をレンダリングするのより前です。裏の世界より表の世界のほうが素早い反映が必要なのでこれは妥当ですね。

# `useTransition`はトランジションを定義する

改めて考えてみると、`useTransition`は「**トランジションを定義する**」という意味のフックであると見なすことができます。そして、`useTransition`から返ってくる`startTransition`関数は**その**トランジションを開始する関数であり、`isPending`フラグは**その**トランジションが実行中かどうかを表すフラグなのです。`useTransition`を使わなくてもReactからデフォルトでエクスポートされている`startTransition`は、Reactがあらかじめ用意してくれた、言わば“デフォルトトランジション”を実行するものだと解釈できます。

ただし、どうやら複数のトランジションを定義しても「裏の世界」はひとつだけのようです。複数の裏の世界を同時に作ることはできません。複数のトランジションを同時に実行したとしても、それらはひとつの「裏の世界」に合流します。このことは、次のように`App`を修正してみれば確かめることができます。

```diff tsx
 function App() {
   const [counter, setCounter] = useState(0);
   const time = useTime();
   const [isPending, startTransition] = useTransition();
+  const [isPending2, startTransition2] = useTransition();
   return (
     <div className="text-center">
       <h1 className="text-2xl">React App!</h1>
       <p className={"tabular-nums" + (isPending ? " text-blue-700" : "")}>
         🕒 {time}
       </p>
       <Suspense fallback={<p>Loading...</p>}>
         <ShowData dataKey={counter} />
       </Suspense>
       <p>
         <button
           className="border p-1"
           onClick={() => {
             startTransition(() => {
               setCounter((c) => c + 1);
             });
+            startTransition2(() => {
+              setCounter((c) => c + 5);
+            });
           }}
         >
           Counter is {counter}
         </button>
       </p>
     </div>
   );
 }
```

この例は1つ目のトランジションで`(c) => c + 1`を実行し、2つ目のトランジションで`(c) => c + 5`を実行します。もし2つの裏の世界ができるならば、counter=1の世界とcounter=5の世界ができるはずです。しかし実際に`console.log`などを追加して試してみると、counter=6の裏の世界のみが観測されるはずです。これは、2つのトランジションが同じ裏の世界に合流していることを示しています。裏の世界は1つだけなのです。

このように、`useTransition`は、あるトランジションが実行中かどうかに紐づいたフラグを提供してくれるという点が主な有用性です。複数のトランジションを同時に走らせてもあまり旨味がありませんが、異なるステート更新に異なるトランジションを使うことで、フラグを見て今どのステート更新が走っているのか知ることができるでしょう。