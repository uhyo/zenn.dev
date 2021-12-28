---
title: "Render-as-you-fetchパターンの実装"
---

前章ではSuspenseに対応した非同期データ取得を実装できました。実はあそこで実装されたのは、React Queryのドキュメントの言葉を借りれば**Fetch-on-render**パターンです。つまり、`useData`を使用するコンポーネントがレンダリングされた時点でデータの取得が始まるということです。

一方で、当初Suspense for data fetchingが発表された時に喧伝されていたSuspenseの利点として、**Render-as-you-fetch**パターンが可能であるという点がありました。そこで、次はこちらのパターンの実装にもチャレンジしてみましょう。

この章の内容を反映したブランチは`chapter/render-as-you-fetch`です。

# Render-as-you-fetchパターンとは

Render-as-you-fetchパターンとは、色々なデータが取得されるにつれて、その部分を表示するコンポーネントがレンダリングされていくという挙動のことです。

つまり、このパターンではデータ取得を行う主体と、それを表示するコンポーネント（つまりローディング中はサスペンドするコンポーネント）が別々になるということです。これは前章の`useData`のようなフックでは達成できません。

これを実現するには、レンダリングを担当するコンポーネントは「ローディング中のデータを受け取る」という機能が必要です。つまり、「ローディング中のデータ」という概念を表す値が必要になってきます。お察しのとおり、JavaScriptでこれを担当するのは**Promise**オブジェクトなのですが、実は今回はPromiseでは機能が不足しています。というのも、解決されたPromiseから中身を取得するには`then`（または内部で`then`を利用している`await`）が必要であり、必ず非同期的に値を取得することになるのです。一方で、コンポーネントのレンダリングは同期関数なので、同期的に取得された値を取り出せることが必要です。

そこで、Promiseをラップして、データ取得済の場合は同期的に値を取り出せるオブジェクトを作ってみましょう。これは**Loadable**と名付けます（Recoilに倣った命名です）。

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

このクラスは`new Loadable(なんらかのPromise)`のように使います。Loadableの内部でステート（`#state`）が管理され、Promiseが解決（成功または失敗）するとそのことを記録します。Promise本体とは別に管理することで、必要な場合に同期的に内容を取得できるようにします。その際に使うのが`getOrThrrow`メソッドで、Suspenseでの利用を見越した実装になっています。

`getOrThrow`メソッドはラップされたPromiseが成功裏に解決済の場合はその値を返します。それ以外の場合、Promiseが失敗した場合はそのエラーを投げます。そして、まだ解決していない場合はPromiseを投げます。

そして、このようなオブジェクトを「ローディング中のデータ」としてコンポーネント間でやり取りすることでrender-as-you-fetchパターンが実現できます。

# データを受け取ってサスペンドするコンポーネントを実装する

`Loadable`ができたら、次は「`Loadable`を受け取ってローディング中はサスペンドするコンポーネント」を実装しましょう。それができればもう簡単なrender-as-you-fetchの完成です。

そして、そのコンポーネントはこれ以上なく簡単に実装できるはずです。具体的には、次のようにします。

```tsx
const DataLoader: React.VFC<{
  data: Loadable<string>;
}> = ({ data }) => {
  const value = data.getOrThrow();
  return (
    <div>
      <div>Data is {value}</div>
    </div>
  );
};
```

Suspense関係のロジック（Promiseをthrowする）が`getOrThrow`の中に押し込められていることによって、コンポーネントの実装はとても簡単になりました。抽象化の力というやつですね。

今回は`DataLoader`の外側でデータの取得を用意しないといけないので、`App`で用意しましょう。また、render-as-you-fetchの効果が分かりやすいように複数の`DataLoader`を使用します。

```tsx
function App() {
  const [data1] = useState(() => new Loadable(fetchData1()));
  const [data2] = useState(() => new Loadable(fetchData1()));
  const [data3] = useState(() => new Loadable(fetchData1()));
  return (
    <div className="text-center">
      <h1 className="text-2xl">React App!</h1>
      <Suspense fallback={<p>Loading...</p>}>
        <DataLoader data={data1} />
      </Suspense>
      <Suspense fallback={<p>Loading...</p>}>
        <DataLoader data={data2} />
      </Suspense>
      <Suspense fallback={<p>Loading...</p>}>
        <DataLoader data={data3} />
      </Suspense>
    </div>
  );
}
```

ポイントは、3つのデータが取得でき次第他を待たずに表示してほしいので、それぞれの`DataLoader`を別々の`Suspense`で囲んでいるという点です。こうすることで、最初は「Loading...」が3つ表示されて、1秒後に次のように3つのデータが表示されることが確認できます。

```
Data is Hello, 629
Data is Hello, 273
Data is Hello, 574
```

また、全部データ取得完了が1秒後だと3つの別々の`Suspense`にした効果が分かりにくいので、次のように`fetchData1`を改造して取得完了までの時間をランダムにしてみましょう。

```ts
async function fetchData1(): Promise<string> {
  await sleep(Math.floor(Math.random() * 1000));
  return `Hello, ${(Math.random() * 1000).toFixed(0)}`;
}
```

そうすれば、3つの「Loading...」が別々のタイミングで実際のデータに切り替わることが確認できるでしょう。

これで簡単なrender-as-you-fetchパターンが実装できましたね。このパターンにより、データ取得は1箇所にまとめて（今回は`App`内にまとまっていますね）、それぞれのデータが取得できたタイミングでデータを表示するということをSuspenseを活用しながら実現できました。

このパターンは他にも、1つのデータを複数箇所の`DataLoader`に渡したりといった応用が可能です。Suspense対応のデータを、コンポーネント間でprops（あるいはコンテキストなど）を通じて受け渡すという従来のモデルに乗せて取り扱える点が魅力的ですね。ただし、この方法では副作用が不必要に何回も起こらないようにキャッシュするなどの工夫は別途必要になってきます。

以上でReact Suspenseを手書きで使ってみるハンズオンは終わりです。しかし、この本はSuspenseの基本を触っただけで、他にもSuspenseには奥深い機能が備わっています。具体的には`useTransition`やSuspense Cacheなどです。これらについては、次の機会にご紹介するとしましょう。