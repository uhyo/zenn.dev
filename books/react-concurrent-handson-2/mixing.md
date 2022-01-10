---
title: 表の世界と交差させよう
---

では、次はもっと自体を複雑にしていきましょう。表の世界でもステート更新が起きるようにしてみます。

# 時計を設置する

さっそくですが、こんなカスタムフックを用意してみました。

```ts
const formatter = Intl.DateTimeFormat("ja-JP", {
  hour: "2-digit",
  minute: "2-digit",
  second: "2-digit",
  fractionalSecondDigits: 1,
});

export function useTime() {
  const [timeString, setTimeString] = useState("");
  useEffect(() => {
    const interval = setInterval(() => {
      setTimeString(formatter.format(new Date()));
    }, 100);
    return () => clearInterval(interval);
  });
  return timeString;
}
```

`useTime`は内部でステート（`timeString`）をひとつ保持し、それを0.1秒ごとに現在時刻を表す文字列（`23:07:43.9`のような）で更新していきます。つまり、このフックを使うと0.1秒ごとにステート更新が発生します。

これを`App`に設置してみましょう。

```diff tsx
 function App() {
   const [counter, setCounter] = useState(0);
+  const time = useTime();
   return (
     <div className="text-center">
       <h1 className="text-2xl">React App!</h1>
+      <p className="tabular-nums">🕒 {time}</p>
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

するとアプリの表示は次の画像のようになり、現在時刻の表示が追加されたことが分かります（0.1秒ごとに表示が更新されます）。カスタムフックですから、`useTime`が作ったステートは`App`に属するステートとなります。

![アプリ上に現在時刻の表示が追加された様子](/images/react-concurrent-handson-2/mixing-1.png)

しかし、それはいいのですが、こうすると`ShowData`の部分が何だか不穏な動きをするはずです。`Data for 0 is`の後の表示が定まらず、10回ほどチカチカと数字が変化する様子が観察されるはずです。これは`0`に対するデータの取得が10回ほど走っていることを意味しています。また、「Counter is 0」ボタンを押したあともやはりデータの取得が複数回走ってしまい、1秒後〜2秒後にかけて`ShowData`が表示するデータが変化し続けます。

なお、「Counter is 0」ボタンを押したあとにそのステート更新が反映される（「Counter is 1」になる）のはこれまで通り1秒後ですが、その間も時計は0.1秒ごとに表示が更新され続けているという点にも注目してください。

## ShowDataの挙動を理解する

では、今回トランジション（`setCounter`）とトランジションではない更新（`useTime`）を組み合わせたことでなにが起こったのでしょうか。これを紐解いていきましょう。