---
title: "トランジションの挙動を調べてみよう"
---

トランジションは優先度の低い更新であるため、レンダリングが中止される可能性があると紹介しました。そこで、次はこれを実際に体験してみましょう。

この章の内容を反映済みのコードは`chapter/cancellation`ブランチにあります。

# 下準備: useDataを持ってくる

ちょっと下準備として、前回の本で作った`useData`を持ってきましょう。さすがにグローバル変数を直で扱うサンプルだとこの先がつらいので、キャッシュキーが利用可能な`useData`を用いてこの先のコードを書いていきます。

```ts
const dataMap: Map<string, unknown> = new Map();

export function useData<T>(cacheKey: string, fetch: () => Promise<T>): T {
  const cachedData = dataMap.get(cacheKey) as T | undefined;
  if (cachedData === undefined) {
    throw fetch().then((d) => dataMap.set(cacheKey, d));
  }
  return cachedData;
}
```

そして、`useData`を使うコンポーネントも用意しましょう。`ShowData`という名前にします。propsでキーを受け取ることでそのキャッシュキーのデータを表示するようにします。

```tsx
export const ShowData: React.VFC<{
  dataKey: number;
}> = ({ dataKey }) => {
  const data = useData(`ShowData:${dataKey}`, fetchData1);
  return (
    <p>
      Data for {dataKey} is {data}
    </p>
  );
};
```

`fetchData1`もやはり以前と同じでよいでしょう。

```ts
export async function fetchData1(): Promise<string> {
  await sleep(1000);
  return `Hello, ${(Math.random() * 1000).toFixed(0)}`;
}
```

# トランジション有りと無しの挙動を比較してみよう

今回、`ShowData`に渡す`dataKey`を`App`のステートとして管理してみましょう。そうすることで、トランジション有りと無しの面白い違いが見えてきます。

## トランジション無しの場合

```tsx
function App() {
  const [counter, setCounter] = useState(0);
  return (
    <div className="text-center">
      <h1 className="text-2xl">React App!</h1>
      <Suspense fallback={<p>Loading...</p>}>
        <ShowData dataKey={counter} />
      </Suspense>
      <p>
        <button
          className="border p-1"
          onClick={() => {
            setCounter(counter + 1);
          }}
        >
          Counter is {counter}
        </button>
      </p>
    </div>
  );
}
```

例によって`counter`というステートを用意しており、ボタンを押すと1増えます（`setCounter(c => c + 1)`じゃないの？　と思われたと思いますが、これは伏線なのでひとまず気にしないでください）。そして、`ShowData`に`counter`の値を渡しています。

こうすることで、ボタンを押して`counter`が増えるたびに`ShowData`が参照するキャッシュキーが変わって再びサスペンドし、1秒後にレンダリングが完了する挙動となります。

このページを表示直後は次のような表示となり、`ShowData`はサスペンドしています。

![「Loading...」と「Counter is 0」ボタンが表示されている](/images/react-concurrent-handson-2/cancellation-1.png)

1秒経つと`ShowData`のレンダリングが完了し、`ShowData`の中身が表示されて次のようになります。

![「Data for 0 is Hello, 373」という文字列と「Counter is 0」ボタンが表示されている](/images/react-concurrent-handson-2/cancellation-2.png)

ボタンを押して`counter`の値を増やすたびに`ShowData`はサスペンドし、（今回はトランジションを使っていないので）「Loading...」が再び表示されます。そして、1秒後に`ShowData`の中身が表示されます。例えば、`counter`を6まで増やして1秒経ったあとは次のような表示になります。

![「Data for 6 is Hello, 985」という文字列と「Counter is 6」ボタンが表示されている](/images/react-concurrent-handson-2/cancellation-3.png)

以上はトランジション無しの場合であり、すでに学んだ内容で理解できます。

## トランジション有り

では、`App`のステート更新をトランジションにしてみましょう。

```diff tsx
 function App() {
   const [counter, setCounter] = useState(0);
   return (
     <div className="text-center">
       <h1 className="text-2xl">React App!</h1>
       <Suspense fallback={<p>Loading...</p>}>
         <ShowData dataKey={counter} />
       </Suspense>
       <p>
         <button
           className="border p-1"
           onClick={() => {
+            startTransition(() => {
               setCounter(counter + 1);
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

そして「Counter is 0」ボタンを押してみましょう。すると、直後は画面が何も反応しないはずです。1秒経つと`ShowData`の中身が「Data for 1 is …」に更新され、同時にボタンが「Counter is 1」になります。

つまり、ここで**トランジションの結果として起こるレンダリングが遅延された**のです。遅延が起こった直接的な理由は言わずもがな、`ShowData`がサスペンドしたことです。そして、トランジションの特異な点は、**サスペンドの影響が`<Suspense>`の外まで漏れ出ている**点にあります。というのも、`counter`は`Suspense`の外にあるステートで、`Suspense`の外でも使われているのに、そこのレンダリングも遅延されていますね。

トランジションの結果としてサスペンドが発生したことで、その影響は`Suspense`の中にとどまらず、そもそもトランジションの画面への反映そのものが遅延されました。これがトランジションの基本的な挙動です。このような挙動は、トランジションが優先度の低いステート更新であることにより正当化されます。サスペンドしたならそれが完了するまでトランジション全体を遅延しちゃおうというアイデアですね。

ちなみに、「Counter is 0」の状態でボタンを連打した場合、どれだけ押しても1秒後に「Counter is 1」になります。押した数だけカウンターが増えたりはしません。これは、「Counter is 0」が表示されている間はずっと`counter`は0の世界が見えているので、何度押しても`setCounter(1)`されているからであると理解できます。

# トランジションとレンダリングの一貫性

Reactが保証するレンダリングの一貫性（整合性と言ってもいいかもしれません）は、Reactがレンダリングに対して保証してくれる性質です。すなわち、レンダリング結果はある時点のステートから全体が生成されるのであり、異なる時点のステートからの結果がひとつのレンダリング結果に混ざることはありません。

上で見たように、トランジションがある場合とない場合では異なる方法でこの一貫性が保証されています。

トランジションが無い場合は、ステートの更新を即座に画面に反映することが優先されます。ただし、サスペンドした部分は新しいステートに対応した表示ができないので`<Suspense>`により提供されるフォールバック表示になります。

一方、トランジションがある場合は、サスペンドした部分をフォールバックせずに古い表示（＝以前のステートのレンダリング結果）のまま残すことが優先されます。このとき一貫性を保つために、サスペンドした部分以外も古い表示のままになるのです。

このように、トランジションに関わるレンダリングの挙動というのは、レンダリングの一貫性保証という観点から理解すると分かりやすいかもしれません。

ところで、この本のタイトルにもなっている「**世界を分岐させる**」という意味深なワードについてここで触れておきましょう。実は、ここでいう世界というのは「ある時点のステート全て」を意味します。つまり、ここで述べているレンダリングの一貫性保証というのは、レンダリング結果は必ずひとつの世界に由来するのであって、異なる世界からの結果が混ざって表示されることは無いということになります。

そして、世界の分岐というのは、異なる世界を複数同時にReactが管理するという意味です。これはトランジションにより初めて発生する挙動であり、トランジションが無い場合は世界の分岐は起こりません。トランジションが無い場合、ステートの更新により即座に世界が書き換わり、アプリ全体が新しい状態に移行し、以前の世界の状態の痕跡は残りません。この意味で、世界はひとつだけです。

ということで、次の章では世界の分岐を垣間見てみることにしましょう。
