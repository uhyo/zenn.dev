---
title: "コンポーネントの外部にデータを持とう"
---

前章ではコンポーネント内のuseStateにデータを保持することで何とかSuspense対応の非同期データ取得を実装しましたが、コンポーネントを初手でサスペンドさせられないなど問題がありました。次はこの問題を克服しなければなりません。

そのためには、**コンポーネントの外部にデータを持つ**ことが必要です。初回レンダリングでサスペンドするとコンポーネントがレンダリングされた痕跡を消されてしまうので、コンポーネントの外部にデータを持つことが必然的に必要になります。

この章の内容を反映したブランチは`chapter/data-fetching-2`です。

# 原始的なデータ取得コンポーネント

ということで、最も原始的な方法でこれをやってみましょう。そう、**グローバル変数**です[^note_module_scope]。

[^note_module_scope]: 正確にはグローバルスコープではなくモジュールスコープの変数ですが。

```tsx
let data: string | undefined;

export const DataLoader: React.VFC = () => {
  // dataがまだ無ければローディングを開始する
  if (data === null) {
    throw fetchData1().then((d) => (data = d));
  }
  // データがあればそれを表示
  return (
    <div>
      <div>Data is {data}</div>
    </div>
  );
};
```

このように`DataLoader`を定義して画面を表示すれば、最初に「Loading...」と表示されて1秒後に「Data is Hello, 464」のように変化するはずです。コンポーネント自体の挙動としては、これが理想的な姿ですね。

「React使いがみんなステート管理で苦労してるのにグローバル変数とかバカにしてるのか？」と思う方も多いでしょうが、実はそこまで的外れな方向性でもありません。そもそも、Reduxなどを使ってシングルトンのストアを保持していたらだいたいグローバル変数のようなものです。ということで、このままもう少し進んでみましょう。

# データ取得フックを作る

まず、`DataLoader`の中に書かれているデータ取得ロジックをフックにしてみましょう。コンポーネントのロジックはフックに書くというのが最近のReactにおけるコンポーネント設計の基本です。

```ts
let data: string | undefined;

function useData1(): string {
  // dataがまだ無ければローディングを開始する
  if (data === undefined) {
    throw fetchData1().then((d) => (data = d));
  }
  return data;
}
```

このフックを使う側はこうなります。

```tsx
const DataLoader: React.VFC = () => {
  const data = useData1();
  return (
    <div>
      <div>Data is {data}</div>
    </div>
  );
};
```

双方ともとてもシンプルになりましたね。これがフックの威力です。特に注目すべき点としては、`useData1`の返り値の型が`string`となっていて、`null`がインターフェースから隠蔽されている点です。詳しくは以下の記事で解説していますが、throwの活用によってインターフェースをシンプルにできるのがSuspenseの面白い点です。

https://qiita.com/uhyo/items/255760315ca61544fe33

また、`DataLoader`の側も、サスペンドする可能性があることが見えにくくなっているものの、ローディング中という状態を意識しないコードにすることができました。サスペンドの可能性が見えにくいという問題点は、コンポーネントはサスペンドするのが普通というように我々のマインドセットが変わっていくことで解消されていくのではないかというのが個人的な予想です。

# 複数のデータを保存できるようにする

現状のコードではデータがグローバルな変数に保存されているため、複数のコンポーネントが`useData1`フックを使用したらデータが共有されてしまいます。データをキャッシュしてくれるという見方をすれば悪くない気もしますが、やはり不便だし気乗りしません。

ということで、**キャッシュキー**を指定できるようにしてみましょう。異なるキーを`useData1`を使う側から指定することで、データが共有されないようにします。

```ts
const dataMap: Map<string, string> = new Map();

function useData1(cacheKey: string): string {
  const cachedData = dataMap.get(cacheKey);
  if (cachedData === undefined) {
    throw fetchData1().then((d) => dataMap.set(cacheKey, d));
  }
  return cachedData;
}
```

こうすれば、異なるキャッシュキーを指定することで別々に読み込むことが可能になります。使う側はこのようにします。

```tsx
const DataLoader1: React.VFC = () => {
  const data = useData1("DataLoader1");
  return (
    <div>
      <div>Data is {data}</div>
    </div>
  );
};

const DataLoader2: React.VFC = () => {
  const data = useData1("DataLoader2");
  return (
    <div>
      <div>Data is {data}</div>
    </div>
  );
};
```

```tsx
function App() {
  return (
    <div className="text-center">
      <h1 className="text-2xl">React App!</h1>
      <Suspense fallback={<p>Loading...</p>}>
        <DataLoader1 />
        <DataLoader2 />
      </Suspense>
    </div>
  );
}
```

こうすると2つの別々のデータが表示されるようになりましたね。キャッシュキーというグローバル管理な値が残っているところが気になりますが、コンポーネントごとに異なるデータを取得できるようになり、進歩しています。

# フックを汎用化する

ところで、`useData1`で取得できるのは`fetchData1`のデータのみです。ここを汎用化してデータ取得関数も外から渡すようにした`useData`を作ることはできないでしょうか。

答えは、一応作ることができます。一応というのは、単純な作りにするとTypeScriptの型部分がちょっと微妙になるからです。汎用化した`useData`は次のようになります。

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

微妙な部分というのは`as`が使われているところで、どのキャッシュキーに対してどの型が入るかあらかじめ分からないのでこうなっています。また、この`as`は実際に危険な`as`です。というのも、このような単純な実装だと複数箇所で別の型で同じキャッシュキーを使われたりすると壊れてしまいます。今回は、とりあえずそこの所はご愛嬌としましょう。

ともあれこの`useData`を使うとコンポーネント側はこのように書き換えられます。

```tsx
const DataLoader1: React.VFC = () => {
  const data = useData("DataLoader1", fetchData1);
  return (
    <div>
      <div>Data is {data}</div>
    </div>
  );
};

const DataLoader2: React.VFC = () => {
  const data = useData("DataLoader2", fetchData1);
  return (
    <div>
      <div>Data is {data}</div>
    </div>
  );
};
```

このように書き換えてもこれまでと同様に動作するはずです。いい感じですね。

ところで、このフックを使っているところをよく見てみましょう。

👀

```ts
const data = useData("DataLoader1", fetchData1);
```

🤔 うーん、何かと似ているような？

🤔💭

![React Queryのロゴ](/images/react-concurrent-handson/react-query-logo.png)

> ```ts
> const { isLoading, error, data } = useQuery('repoData', () =>
>   fetch('https://api.github.com/repos/tannerlinsley/react-query').then(
>     res => res.json()
>   ))
> ```
>
> [Overview | React Query | TanStack](https://react-query.tanstack.com/overview) からインデント等を調整して引用

🤔💡

おお、`useData`のインターフェースが、非同期データフェッチングライブラリの一つとして知られるReact Queryが提供する`useQuery`フックとだいたい同じですね。[React QueryのSuspense対応を説明するページ](https://react-query.tanstack.com/guides/suspense)ではSuspense対応モードでは`isLoading`や`error`といったものは必要ないことも説明されていますから、Suspense対応の前提ではReact Queryと今回作ったフックはインターフェースが同じと言えます。

つまり、以上で原始的なReact Queryを実装することができたということです。おめでとうございます。