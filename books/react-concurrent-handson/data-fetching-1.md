---
title: "非同期データ取得を実装してみよう"
---

前章ではコンポーネントのサスペンドや、サスペンドからの復帰の様子を観察しました。しかし、これまでのコンポーネントはランダムにサスペンドするとか、Suspenseを使いこなしている感が全くしない実験的な例でしたね。

ここからは、Suspenseを本来の用途で使うことを目指します。ここからがハンズオンの本番です。

まずは非同期データ取得を模したデータ取得関数を定義しておきましょう。この関数は1秒後に「Hello, 123」のようなランダムな数字を含む文字列を返します。

```tsx
async function fetchData1(): Promise<string> {
  await sleep(1000);
  return `Hello, ${(Math.random() * 1000).toFixed(0)}`;
}
```

# `useState`を使ってみる（失敗例）

データのローディングを担当するコンポーネントに求められる挙動は次の通りです。

- 1回目のレンダリングではデータのローディングを開始し、ローディングが完了したら解決されるPromiseをthrowする。
- 2回目のレンダリングでは、ローディングが完了したデータを表示する。

このことからわかるように、ローディングされたデータをどこかに保持しておく必要があります。おそらく最初に思いつくのが`useState`でステートに保持しておくことでしょう。

しかし、残念ながらこれはうまくいきません。なぜうまくいかないのかを理解するために、とりあえず実装してみましょう。想定される実装はこんな感じです。

```tsx
const DataLoader: React.VFC = () => {
  const [data, setData] = useState<string | null>(null);
  // dataがまだ無ければローディングを開始する
  if (data === null) {
    throw fetchData1().then(setData);
  }
  // データがあればそれを表示
  return <div>Data is {data}</div>;
};
```

使う側はこんな感じでしょう。

```tsx
function App() {
  return (
    <div className="text-center">
      <h1 className="text-2xl">React App!</h1>
      <Suspense fallback={<p>Loading...</p>}>
        <DataLoader />
      </Suspense>
    </div>
  );
}
```

これを実行してみると、残念ながら画面は「Loading...」のままです。これは、`DataLoader`コンポーネントがサスペンドしたままだということを意味しています。また、次のようなWarningが表示されます。何やら、「**まだマウントされていないコンポーネントのステートを更新することはできませんよ**」ということです。最後に書いてあるuseEffectを使えというのは今回は当てはまらないので気にしなくて構いません。

> Warning: Can't perform a React state update on a component that hasn't mounted yet. This indicates that you have a side-effect in your render function that asynchronously later calls tries to update the component. Move this work to useEffect instead.

コンポーネントのサスペンドというのは日本語に直すと「コンポーネントの（レンダリングの）中断」ということです。つまり、そもそもコンポーネントのレンダリングが成功していないので、ステートの記憶領域が用意されていないのです。つまり、コンポーネントがサスペンドした場合には、（そのコンポーネントから投げられたPromiseを除いては）そのコンポーネントのレンダリングを試みたという記録が歴史から抹消されます。よく「Reactコンポーネントのレンダリング中に副作用を起こしてはいけない」と言われますが、その実際的な理由の一端がこれです。レンダリングは無かったことにされたのに副作用だけ残ってしまう恐れがあるので、やってはいけないのです。

要するに、`DataLoader`が再レンダリングされるというのはあくまでReactのランタイムが再びレンダリングを試みるということであって、`DataLoader`にとっては毎回が初回レンダリングなのです。そのため、何度Reactが再レンダリングを試みても`data`は毎回nullであり、永遠に`DataLoader`はレンダリングを成功させることができないのです。調べてみれば、`DataLoader`が1秒ごとにレンダリングされて毎回`data`がnullであることが観察できるはずです。

ちなみに、`useState`だけでなく`useRef`を使ってもコンポーネント内にデータを保持することはできません。フック用の記憶領域はすべてレンダリングが完了しないと用意されないのです。

# ならばマウント後にサスペンドすれば？（一応成功）

ところで、ステートの記憶領域がないことが問題ならば、それを用意してからサスペンドさせるとどうなるでしょうか。つまり`DataLoader`が初手でサスペンドするのではなく、ボタンを押したらサスペンドするようにしてみます。

```tsx
const DataLoader: React.VFC = () => {
  const [loading, setLoading] = useState(false);
  const [data, setData] = useState<string | null>(null);
  // ローディングフラグが立っていてdataがまだ無ければローディングを開始する
  if (loading && data === null) {
    throw fetchData1().then(setData);
  }
  // データがあればそれを表示
  return (
    <div>
      <div>Data is {data}</div>
      <button className="border p-1" onClick={() => setLoading(true)}>
        load
      </button>
    </div>
  );
};
```

これを試してみると、最初は「data is」と表示されます（nullはレンダリングされないので）。loadボタンを押すことで「Loading...」表示になり、1秒後に「Data is Hello, 520」のような表示になります。

おめでとうございます！　これでSuspense対応非同期データローディングコンポーネントができました。

ただ、実はこれは**2つの理由からお勧めしません**。

一つは、データをロードするコンポーネントなのに初手でサスペンドしないというのはおかしいという点です。Suspenseの利点は「コンポーネントのレンダリングが成功したならばデータの表示にも成功している（コンポーネントがローディング中の表示の責務を負わない）」というようにしてコンポーネントの責務を単純化できる点にあるのに、初手でサスペンドできないとなるとその利点が失われています。

もう一つの理由は、実は上のコードで起こっている現象がいわば「**サスペンドキャンセル**」のようなものであり、実はReactのサスペンドリトライ機構に乗っていないからです。

後者がどういうことかを理解するために、コンポーネントを次のように変えてみましょう。

```diff tsx
 const DataLoader: React.VFC = () => {
   const [loading, setLoading] = useState(false);
   const [data, setData] = useState<string | null>(null);
   // ローディングフラグが立っていてdataがまだ無ければローディングを開始する
   if (loading && data === null) {
+    sleep(500).then(() => setData("boom!"));
     throw fetchData1().then(setData);
   }
   // データがあればそれを表示
   return (
     <div>
       <div>Data is {data}</div>
       <button className="border p-1" onClick={() => setLoading(true)}>
         load
       </button>
     </div>
   );
 };
```

Promiseをthrowする前に、500ミリ秒後の`setData`を仕込みました。この状態でloadボタンを押すと次の挙動になります。

- ボタンを押すと「Loading...」表示になる。
- 500ミリ秒後に「Data is boom!」のように表示される。
- さらに500ミリ秒後に「Data is Hello, 520」のような表示になる。

このように、`fetchData1()`には1秒かかるので1秒後まで`DataLoader`がサスペンドしたままであると思いきや、実はPromise解決によるリトライより前にステートが変更されると、`DataLoader`コンポーネントはその時点で再レンダリングされます。そして、`setData`により`data`に値が入ったことで今回のレンダリング結果はサスペンドとはならず、その結果が「Data is boom!」という表示だったのです。一度再レンダリングされた時点で、Promise解決によるリトライのスケジュールはキャンセルされます。

つまり、すでにマウント済みのコンポーネントに関しては、Promise解決による再レンダリングはあくまでサスペンド解除の手段であり、他の手段（ステート更新）により再レンダリングを引き起こしてサスペンドを解除させることもできるのです。

では、`boom!`を追加する前のコードをもう一度見てみましょう。

```tsx
const DataLoader: React.VFC = () => {
  const [loading, setLoading] = useState(false);
  const [data, setData] = useState<string | null>(null);
  // ローディングフラグが立っていてdataがまだ無ければローディングを開始する
  if (loading && data === null) {
    throw fetchData1().then(setData);
  }
  // データがあればそれを表示
  return (
    <div>
      <div>Data is {data}</div>
      <button className="border p-1" onClick={() => setLoading(true)}>
        load
      </button>
    </div>
  );
};
```

この場合も、実は「Promise解決による再レンダリング」より前に「setDataによる再レンダリング」が起こっています。`fetchData1().then(setData)`の返り値であるPromise（throwされたPromise）が解決されるよりも、`setData`が呼び出される方が先だからです。

これでも期待通りに動くことは動くのですが、「投げたPromiseが解決されたら再レンダリングされる」という分かりやすいモデルから外れて「とにかくサスペンドの意思を伝えるためにPromiseを投げる」という挙動になっています。最悪、永遠に解決されないPromiseを投げつつそれとは別に1秒後に`setData`を呼び出してもいいわけです。所詮はローレベルAPIなので動けばそれでいいという側面もありつつ、意図が分かりにくいコードはよろしくないでしょう。

ということで、次章ではよりよいサスペンド方法を考察していきます。結論を先取りしてしまうと、何らかの手段でコンポーネントの外にデータを持つことが必要になってきます。

# 再レンダリングでサスペンドしてもやはり歴史は消される

ところで、コンポーネントのレンダリングがサスペンドした場合、レンダリングを試みたという事実が歴史から抹消されるのでした。これはすでにレンダリングされたコンポーネントが再レンダリング時にサスペンドした場合も例外ではありません。この場合、**その再レンダリング中にコンポーネントの記憶領域に書き込まれたものがロールバックされます**。

具体的には、useStateのステート（レンダリング中にステート更新を起こすのはよろしくありませんが）やuseMemoのメモ化結果などです。後者は普通にあり得るシチュエーションなので、これを観察してみましょう。

```diff tsx
 export const DataLoader: React.VFC = () => {
   const [loading, setLoading] = useState(false);
   const [data, setData] = useState<string | null>(null);

+  const _ = useMemo(() => {
+    if (loading) {
+      console.log("loading is true");
+    }
+    return 1;
+  }, [loading]);

   // ローディングフラグが立っていてdataがまだ無ければローディングを開始する
   if (loading && data === null) {
     throw fetchData1().then(setData);
   }
   // データがあればそれを表示
   return (
     <div>
       <div>Data is {data}</div>
       <button className="border p-1" onClick={() => setLoading(true)}>
         load
       </button>
     </div>
   );
 };
```

意味のないuseMemoを追加してみました。`loading`がtrueの際にログを出力するようになっています。この状態でloadボタンを押すと、「loading is true」というログは2回出力されます。ボタンを押した直後に1回、1秒後のデータ取得完了時に1回です。

これは、ボタンを押した直後に`loading`がtrueの状態でレンダリングが行われてuseMemoの関数が呼び出されたものの、そのレンダリングはサスペンドしたため、メモ化内容（useMemoの結果）が捨てられたことを意味しています。そのため、1秒後の再レンダリングでは再度useMemoの関数が呼び出されたのです。console.logを用いたデバッグはよく行われますが、サスペンドが絡んだコンポーネントをデバッグする際はこういったことにも気をつける必要があります。

# まとめ

この章で学んだことをまとめました。

- 最初のレンダリングでサスペンドしたコンポーネントはステートなどを保持できない。
- マウント済みのコンポーネントがサスペンドした場合、Promise解決による再レンダリング以外にステート更新などによる再レンダリングでもサスペンドが解除される。
- 再レンダリング時にサスペンドした場合、その再レンダリングの最中にコンポーネントの記憶領域に書き込まれたものは失われる。