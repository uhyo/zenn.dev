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
  const [time, setTime] = useState("");
  useEffect(() => {
    const interval = setInterval(() => {
      setTime(formatter.format(new Date()));
    }, 100);
    return () => clearInterval(interval);
  });
  return time;
}
```

`useTime`は内部でステート（`time`）をひとつ保持し、それを0.1秒ごとに現在時刻を表す文字列（`23:07:43.9`のような）で更新していきます。つまり、このフックを使うと0.1秒ごとにステート更新が発生します。

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

以下の図は、前章に出てきた図に非トランジションの更新（`setTime`）を付け足したものです。最初は counter=0 の状態からボタンを押した（`setCounter(c => c + 1)`した）ところからスタートし、定期的に`setTime`が行われています。

![2種類の更新によるステート変化の模式図](/images/react-concurrent-handson-2/mixing-2.png)

非トランジションの更新（`setTime`）は、トランジションではないということはすぐ反映されなければならないということです。そのため、表の世界のステートに対して反映されます。このとき、トランジションの更新（`setCounter`）は表の世界に未反映のまま進みます。よって、表の世界は counter=0 のままtimeが更新されていきます。

ただし、`setTime`の更新は裏の世界にも即座に適用されます。その結果、裏の世界では counter=1 の状態でtimeが更新されていきます。このように、世界が分岐されていることで、ひとつのステートの更新が複数の世界に複製されて適用されるのです。

ステートが更新されるということは再レンダリングするということですから、裏の世界のレンダリングも0.1秒に1回行われます。そのたびにサスペンドが発生しますから、図の右側にあるように、1秒後から2秒後にかけて複数回裏の世界から表の世界への反映が行われています。前章では「1度サスペンドが完了したらそれ以降は裏の世界から表の世界に反映されない」のような説明をしましたが、今回のケースでは各サスペンドでtimeが変化しているために別々のステートに対するレンダリングであると見なされ、各サスペンドの結果が0.1秒間隔で全部表の世界にやってくることになります。

これが今回起こっていることの全容でした。

## useDataを修正しよう

ところで、先ほどの例でData for 1 is ... の数字が目まぐるしく変化する事象は、counter=1 に対するサスペンドが発生するごとに別々にデータの取得が行われたからでした。説明のためにはちょうど良かったものの、これは実用的ではありません。すでに counter=1 に対するデータ取得が走っているのであればそれを再利用したいところです。

ということで、そのようにちょっと`useData`を修正して挙動を見てみましょう。

具体的には、前回の本で出てきた`Loadable`を活用します。ただし、新しく`static newAndGetPromise`を追加しておきます。これは`new Loadable`の代わりに使用することで`Loadable`の内部に生成されたPromiseも一緒に返してもらえる関数です。

```ts
type LoadableState<T> =
  | {
      status: "pending";
      promise: Promise<T>;
    }
  | {
      status: "fulfilled";
      data: T;
    }
  | {
      status: "rejected";
      error: unknown;
    };

export class Loadable<T> {
  #state: LoadableState<T>;
  constructor(promise: Promise<T>) {
    this.#state = {
      status: "pending",
      promise: promise.then(
        (data) => {
          this.#state = {
            status: "fulfilled",
            data,
          };
          return data;
        },
        (error) => {
          this.#state = {
            status: "rejected",
            error,
          };
          throw error;
        }
      ),
    };
  }

  static newAndGetPromise<T>(promise: Promise<T>): [Loadable<T>, Promise<T>] {
    const result = new Loadable(promise);
    if (result.#state.status !== "pending") {
      throw new Error("Unreachable");
    }
    return [result, result.#state.promise];
  }

  getOrThrow(): T {
    switch (this.#state.status) {
      case "pending":
        throw this.#state.promise;
      case "fulfilled":
        return this.#state.data;
      case "rejected":
        throw this.#state.error;
    }
  }
}
```

これを用いて、`useData`をこんな感じに修正してみましょう。

```ts
const dataMap: Map<string, Loadable<unknown>> = new Map();

export function useData<T>(cacheKey: string, fetch: () => Promise<T>): T {
  const cachedData = dataMap.get(cacheKey) as Loadable<T> | undefined;
  if (cachedData === undefined) {
    const [loadable, promise] = Loadable.newAndGetPromise(fetch());
    dataMap.set(cacheKey, loadable);
    throw promise;
  }
  return cachedData.getOrThrow();
}
```

従来はデータ取得が完了してから`dataMap`にエントリが作られていましたが、このように修正することで取得中にすでに`Loadable`が入っているようになります。これで同じキャッシュキーに対して複数回フェッチすることを避けられますね。

このように修正してからもう一度アプリの挙動を確認してみると、意図通りの挙動になっているはずです。時計は動きっぱなしのまま、ボタンを押すと1秒後に画面が更新されてデータが取得されます。修正後のステート更新・サスペンドの挙動はこの図のようになっています。

![useData改良後のステート更新の模式図](/images/react-concurrent-handson-2/mixing-3.png)

裏の世界のレンダリングとサスペンドが0.1秒に1回起こる点は変わりませんが、`useData`の改良によってそれらが全部同じ`Loadable`（の中のPromise）を見るようになったことで最初のサスペンド（データ取得開始）から1秒後に一斉に完了します。そのため、表の世界への反映は1回となります。