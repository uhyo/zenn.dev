---
title: "エラーハンドリング"
---

一般に、非同期処理には**エラー**が付きものです。ネットワークエラーやサーバーエラーなど、さまざまな原因で非同期処理が失敗する可能性があります。そのため、Promiseにも「失敗 (reject)」を扱う機能が備わっています。

この章では、Suspenseにおけるエラーハンドリングについて解説します。

## Error Boundaryとは

Reactには、エラーハンドリングの仕組みも組み込まれています。それが**Error Boundary**です。Error Boundaryは、コンポーネントツリー内で発生したエラーをキャッチし、フォールバックUIを表示するための仕組みです。Error Boundary自体はSuspenseよりも昔から存在しており、次のようなレンダリング中のエラーをキャッチできる仕組みです。

```tsx
<MyErrorBoundary>
  {/* この中で発生したエラーをMyErrorBoundaryがキャッチ */}
  <MyComponent />
</MyErrorBoundary>

// エラーの例
const MyComponent = (): React.FC => {
  if (isError()) {
    throw new Error("ぎゃ～～～～～！！");
  }
  return <div>正常なコンテンツ</div>;
};
```

このように、Reactコンポーネントはこのように、レンダリング中に異常があることを表すために`throw`でエラーを投げることができます。Error Boundaryは、そのようにして投げられたエラーをキャッチし、フォールバックUIを表示します。

### Suspenseとの共通点

SuspenseとError Boundaryはとても似ています。順番としてはError Boundaryが先なので、Suspenseがそれと似た仕組みを採用したということになります。

SuspenseもError Boundaryも、内部のコンポーネントが「**レンダリング失敗**」という状況に陥る点が共通しています。Suspenseの場合は「非同期処理が完了していない」という状況、Error Boundaryの場合は「エラーが発生した」という状況です。

どちらも、その時点で中身をレンダリングすることは不可能です。関数コンポーネントであれば、ちゃんとJSXを`return`できていないのですから、何をレンダリングしていいのか分かりません。

そのため、両者ともに**フォールバックUIを表示する**というアプローチを取っています。Suspenseの場合は「ローディング中のUI」、Error Boundaryの場合は「エラー発生時のUI」を表示します。

## Error Boundaryの実装

Error Boundaryは、Suspenseとは異なり、Reactから`ErrorBoundary`コンポーネントが提供されているわけではありません。自分で実装する必要があります。自分で実装するための機能がReactから提供されています。

Error Boundaryを実装するには、クラスコンポーネントを使う必要があります。最小限の実装例を以下に示します。

```tsx
class MyErrorBoundary extends React.Component<
  { fallback: React.ReactNode },
  { hasError: boolean }
> {
  constructor(props: { fallback: React.ReactNode }) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error) {
    // エラーが発生したときにstateを更新する
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

この`MyErrorBoundary`コンポーネントは`hasError: boolean`というステートを持っており、通常はこれがfalseです。この状況では、子コンポーネントをそのままレンダリングします。

しかし、子コンポーネント内でエラーが発生すると、`getDerivedStateFromError`メソッドが呼び出されます。このメソッドはステートオブジェクトを返します。すると、`MyErrorBoundary`のステートがそれに従って更新されます。今回の場合、`hasError`がtrueに更新されます。

すると、再レンダリングにより`MyErrorBoundary`のレンダリング結果がフォールバックUIに切り替わります。

### 再利用可能なコンポーネント

毎回上記のような実装を行うのは大変ですよね。そこで、汎用的に使えるError Boundaryコンポーネントを作成しておいて、それをアプリ内で使いまわすのが一般的なテクニックです。

もしくは、まさにそれを目的としたライブラリを利用するのも良いでしょう。react-error-boundaryはReactの公式ドキュメントからも紹介されています。

https://github.com/bvaughn/react-error-boundary

この本でも、react-error-boundaryから提供される`ErrorBoundary`コンポーネントを用いて解説を進めていきます。

### Error Boundaryのリセット

典型的には、Error Boundaryは**リセット**できる必要があります。リセットというのは、エラーをキャッチしてフォールバックUIを表示している状態から、再度子コンポーネントが表示される状態へ移行することです。

react-error-boundaryの`ErrorBoundary`も、`resetKeys`と`resetErrorBoundary`という2つの仕組みを備えています。

また、Reactの`key`を使い、Error Boundaryを丸ごと再作成することでリセットする方法もあります。

## SuspenseとError Boundaryの関係

では、本題の非同期処理の話です。非同期処理でエラーが発生した場合の処理はどのように書けばいいのでしょうか。お察しのとおり、この場合もError Boundaryを使います。

この本の説明では、Suspenseで非同期処理の結果を得るには以下のようにしました。

```tsx
// useを直接使う場合
const user = use(userPromise);
// jotaiを使っている場合（userAtomはPromise<User>を保持）
const user = useAtomValue(userAtom);
```

このとき、最終的に`userPromise`や`userAtom`が返すPromiseがrejectされると、**`use`や`useAtomValue`の箇所で`throw`されたような挙動になります**。

これはコンポーネントのレンダリングの最中にエラーが発生したのと同じ状況です。そのため、従来と同じようにError Boundaryでキャッチできます。要するに、こういう感じですね。

```tsx
<Suspense fallback={<div>ローディング中...</div>}>
  <ErrorBoundary fallback={<div>エラーが発生しました</div>}>
    <MyComponent />
  </ErrorBoundary>
</Suspense>
```

### Error Boundaryの配置

上の例ではSuspenseの直下に雑にError Boundaryを配置しましたが、実際にはもう少し工夫したほうがいい場面も多いでしょう。

Error Boundaryもまた、その名のとおり**境界**です。内側でエラーが発生したとき、Error Boundaryの外側はエラーの影響から守られます。内側はエラーの犠牲となり、コンテンツが消されてフォールバックUIに置き換えられます。

そのため、「この中だけフォールバックUIに切り替わっても不自然ではない」という範囲で境界を切るのが望ましいでしょう。レンダリングの最中に発生しうるエラーも考慮しながら適切に配置する必要があります。

なお、上の例のようにSuspenseとErrorBoundaryを重ねる場合に、SuspenseとError Boundaryのどちらが内側でどちらが外側かについては特に決まりがありません。どちらでも動作に問題はありません。ただし、エラー発生時にSuspenseがマウントされたままなのか、それともSuspenseごと消されるのかが変わるので、トランジション時の挙動などには差が出ます。それを踏まえて決めるべきです。

### Error Boundary配置の具体例

例えば、テキストにより絞り込み検索を行う場合を考えましょう。検索結果は非同期に取得されるので、エラーの恐れがあるとします。

この場合、検索ボックスと検索結果リストを1つのError Boundaryで囲むのはあまり良くありません。なぜなら、検索結果の取得に失敗した場合に検索ボックスまで消えてしまうのはユーザーにとって不便だからです。

```tsx
// 良くない例
<ErrorBoundary fallback={<div>エラーが発生しました</div>}>
  <SearchBox />
  <Suspense fallback={<div>ローディング中...</div>}>
    <SearchResults />
  </Suspense>
</ErrorBoundary>
```

この場合、検索結果リストだけをError Boundaryで囲むのが良いでしょう。こうすれば、検索結果の取得に失敗しても検索ボックスは残り、ユーザーは再度検索を試みることができます。

```tsx
// 良い例
<SearchBox />
<ErrorBoundary fallback={<div>エラーが発生しました</div>}>
  <Suspense fallback={<div>ローディング中...</div>}>
    <SearchResults />
  </Suspense>
</ErrorBoundary>
```

ちなみに、この例ではError Boundaryが外側、Suspenseが内側に配置されています。これは、失敗時の再試行のステート更新がトランジションである場合に、「古いUI」としてエラー表示が残るのではなく、すぐにフォールバック（ローディング中...）に切り替わることを意図した配置です。

「古い検索結果→新しい検索結果」のときは古いUIが見えていても問題ありませんが、「エラー表示→新しい検索結果」のときに古いUIが見えているのはユーザーにとって不自然だろうという配慮を込めています。

## 非同期処理にどう向き合うか

従来（Suspenseを使わない非同期処理）は、非同期処理のエラーハンドリングは適当に行われがちでした。例えば、`useEffect`の中でfetchするような場合です。

エラーハンドリングをまったくしないわけではなく、例えばエラーが発生したら「エラーが発生しました」みたいなモーダルダイアログを出す仕様はよくあります。

```tsx:従来の実装の例
// contextとかからアプリケーション共通のエラーハンドリング関数を取得
// （これがモーダルダイアログとかを出す）
const handleError = useErrorHandler();
// 取得したユーザー
const [user, setUser] = useState<User | null>(null);
useEffect(() => {
  fetchUser()
    .then((data) => setUser(data))
    .catch((error) => {
      handleError(error);
    });
});
```

それなら適当じゃなくてちゃんとエラーハンドリングしているじゃん、と思われるかもしれません。しかし、上の実装には問題があります。

結局、エラーが発生した場合のUIはどうなるでしょうか。上の実装では、エラーが発生しても`user`は`null`のままです。典型的には、`user`が`null`のままの場合はローディング中のUIを表示するような実装になっているでしょう。

すると、「エラーが発生したらモーダルは出るけど、UIはずっとローディング中のまま」という挙動になります。皆さんも、身に覚えがあるのではないでしょうか。

あるいは、ちゃんと`isLoading`を管理しているからローディング中のUIにはならないかもしれません。それでも、`user`が`null`のときは特に何も描画しないので空白のUIになる、というパターンもありがちです。

もちろん、もともとちゃんとエラー時のUIを設計して実装していた方々もいるでしょう。そのような方々はまったく問題ありません。

しかし、Suspenseは、そうでない方々にも**ちゃんとエラー時のUIに向き合うことを強制します**。エラー時のUIが必要ないなら`fallback={null}`とかでも良いですが、その場合でも「UIのどの範囲が空白になるのか？」といったことは最低限考えてError Boundaryの配置を設計する必要があります。

それを怠ったら、エラー一発でアプリ全体が真っ白です。それはちょっと困りますね。

強制されると言われると嫌なことのように聞こえますが、筆者としてはそこまで悪いことだと思いません。なぜなら、UIの開発者は元々エラーハンドリングについてはきちんと考えて向き合う必要があるからです。やるべきことを半ば強制されるのは、悪いことではありません。

## jotaiを使う場合のエラーハンドリング

先ほど、「エラーが発生したときはエラーモーダルを出す」という例を見せましたね。実際、こういう仕様はよくあります（筆者としては、何でもモーダルダイアログに頼るのではなくちゃんとエラー表示をUIに組み込むべきだと思いますが、それはさておき）。

それは抜きにしても、「エラーが発生したらログを記録する」といった仕様も頻出です。上の例では、`handleError`関数がその役割を担っていたのでしょう。典型的には、`handleError`は裏で`ErrorHandlingContext`みたいなコンテキストから取得されがちです。

では、この本で見てきたようなjotaiによる非同期処理を行っている場合は、どのようにエラーハンドリングを行うべきでしょうか。

そのためにはいくつかの方法があり、場合によって使い分けたり組み合わせたりする必要があります。

### createRootのコールバック

`createRoot`はReactアプリのエントリーポイントとして、特定のDOMノードにReactコンポーネントツリーをマウントする関数です。

```tsx
// 典型的なReactアプリのエントリーポイント
import { createRoot } from "react-dom/client";

const root = createRoot(document.getElementById("root")!);
root.render(<App />);
```

この`createRoot`には、**エラーが発生した場合のコールバックを指定できる仕組み**があります。

これを使うことで、**レンダリング中に発生したエラー（Error Boundaryでキャッチできる種類のエラー）** はすべて一括して処理できます。

`createRoot`に渡すことができるコールバックは3種類あり、異なる種類のエラーを処理できます。

- `onUncaughtError`: レンダリング中に発生し、Error Boundaryでキャッチされなかったエラー
- `onCaughtError`: レンダリング中に発生し、Error Boundaryでキャッチされたエラー
- `onRecoverableError`: レンダリング中に発生したが、Reactにより自動的に回復されたエラー

最後の`onRecoverableError`というのがよく分からないかもしれません。「Reactにより自動的に回復されたエラー」とは、典型的にはハイドレーションミスマッチです。サーバーサイドレンダリングされたHTMLとクライアントサイドでレンダリングされた結果が一致しない場合、Reactは警告を出しつつも自動的に回復してアプリが動きます。

また、Reactはレンダリング中にエラーが発生した場合、そのエラーがレースコンディションによるものであることを疑って再試行を行うことがあります。再試行の結果エラーなくレンダリングできた場合は、「回復できた」と見なされエラーが無かったものとして扱われます。この場合も`onRecoverableError`の対象となります。

`onRecoverableError`対象のエラーはUI上のエラーとは扱われないので、Error Boundaryでキャッチされることもありません。

エラーハンドリングの目的に応じて、上記3つのコールバックを使い分けるといいでしょう。エラーロギングが目的なら3つ全部、エラーモーダルを出すのが目的なら`onUncaughtError`と`onCaughtError`だけといった具合です。

### atom側にエラーハンドリングを組み込む

他のエラーハンドリングの方法として、**atom側にエラーハンドリングの仕組みを組み込む**方法があります。原始的な方法はこうです。

```tsx
const userAtom = atom(async (get) => {
  try {
    const user = await fetchUser();
    return user;
  } catch (error) {
    // エラーハンドリングの処理
    handleError(error);
    // 再throwする（UI側でもこのエラーに対応できるようにするため）
    throw error;
  }
});
```

もう少しシンプルに、エラーハンドリングして再throwまでしてくれる`handleErrorAndRethrow`を用意しておけば、こうも書けます。

```tsx
const userAtom = atom(async (get) => {
  const user = await fetchUser().catch(handleErrorAndRethrow);
  return user;
});
```

もしくは、jotaiらしくユーティリティ関数を作成しても良いでしょう。

```tsx
const userAtom = atomWithErrorHandling(async (get) => {
  const user = await fetchUser();
  return user;
});
```

### データ取得レイヤーにエラーハンドリングを組み込む

もはやjotaiとは関係のない話になりますが、jotaiレイヤーではなく`fetchUser`の内部にエラーハンドリングを組み込む方法もあります。

ちゃんとしたアプリケーションにおけるデータ取得の場合、`fetch`（あるいは昔からのコードベースだと`axios`とか）を直接呼び出すことは少なく、何らかの**データ取得レイヤー**を介して行うことが多いでしょう。「APIクライアント」みたいな名前のモジュールがそれにあたります。

その「APIクライアント」の中にエラーハンドリングの仕組みを組み込む方法もあります。雑な例ですが、イメージとしてはこんな感じです。

```tsx
// apiClient.ts
export async function getRequest(url: string) {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    return await response.json();
  } catch (error) {
    handleError(error);
    throw error;
  }
}
```

## エラーモーダルダイアログを出したい場合はどうするのか

この章では、ありがちなエラー処理としてエラーモーダルに度々言及してきました。上記のように好き勝手な場所から`handleError`を呼び出すことにした場合、エラーモーダルをどう出せばいいのでしょうか。

というのも、古き良き`const handleError = useErrorHandler();`のようなエラー処理コードの場合、その裏側は大体こんな風になっていることが多いです。

```tsx
// ErrorHandlingContext.tsx
const ErrorHandlingContext = createContext<{
  handleError: (error: Error) => void;
}>({
  handleError: () => {},
});

export const ErrorHandlingProvider: React.FC = ({ children }) => {
  const [error, setError] = useState<Error | null>(null);

  const contextValue = useMemo(() => ({
    handleError: (error: Error) => {
      setError(error);
    },
  }), []);

  return (
    <ErrorHandlingContext.Provider value={contextValue}>
      {children}
      {error && <ErrorModal error={error} onClose={() => setError(null)} />}
    </ErrorHandlingContext.Provider>
  );
};

function useErrorHandler() {
  const context = useContext(ErrorHandlingContext);
  return context.handleError;
}
```

つまり、`handleError`がモーダルを出すためにステートを更新しており、そのためにはReactコンポーネントツリーの中で`handleError`を定義する必要があり、だからアプリケーション内では`useContext`を使ってコンテキストから`handleError`を取得しているわけです。

このような構成になっている場合、例えば「atomの中で`handleError`を呼び出せ」と言われても困るわけですね。

モーダルダイアログでなくても、エラー処理の結果としてUIを更新したい場合は同様の問題が発生します。

### 解決策

この問題に対する解決策のひとつは、**エラーハンドリングのUIもjotaiで管理する**ことです。例えば、以下のようにエラーモーダルの表示状態を管理するatomを用意します。

```tsx
const errorAtom = atom<unknown>(null);

// エラーモーダルの表示
const ErrorModalContainer: React.FC = () => {
  const [error, setError] = useAtom(errorAtom);

  if (!error) {
    return null;
  }

  return (
    <ErrorModal
      error={error}
      onClose={() => setError(null)}
    />
  );
};
```

こうすれば、`handleError`関数は`errorAtom`に書き込みを行えばよくなります。実は、一工夫が必要ですがReactの外からでも書き込みを行うことができます。

一工夫とは、**ストアを直接操作する**ことです。jotaiでは、atomのステートが実際に保管されている場所はストアというオブジェクトです。ストアを直接操作すれば、`useAtom`のようなフックを経由しなくてもステートを読み書きできます。

ストアは、`createStore`を使って自分で作ったりもできますが、特にストアを明示的に作ったりしていない場合は**デフォルトのストア**が使用されています。デフォルトのストアは`getDefaultStore()`で取得できます。

```tsx
import { getDefaultStore } from "jotai";

function handleError(error: unknown) {
  const store = getDefaultStore();
  store.set(errorAtom, error);
}
```

これで、atomの外からでもエラーハンドリングのUIを更新できるようになりました。

jotaiに限らず、特にこのようなアプリ全体にかかわるロジックの際は、**何でも無理にReactの中でやる必要はない**ことを意識すると設計の幅が広がります。UIの話だからといって、絶対にステートを`useState`で管理して関数を`useContext`で伝播して……とする必要はありません。敢えてグローバルステートを活用するとシンプルにことが進む場合も多くあります。

## リトライの実装

非同期処理が失敗した場合に、典型的なエラーUIは「エラーが発生しました」と表示するだけではなく「再試行」的な操作を提供することが多いでしょう。これを実装する方法を考えてみましょう。

とはいえ、難しくはありません。atomの計算を再実行させる方法は以前の章で学びました。また、この章の上のほうで`ErrorBoundary`のリセット方法についても説明しました。これらを組み合わせるだけです。

例えば、ユーザー一覧を取得するUIだとすると、おおよそこのような実装になります。

```tsx
// 再実行可能なユーザー一覧atom
const userListAtom = createReloadableAtom(async (get) => {
  // ...
});

// ユーザー一覧の表示
<ErrorBoundary fallbackComponent={UserListErrorFallback}>
  <Suspense fallback={<div>ローディング中...</div>}>
    <UserList />
  </Suspense>
</ErrorBoundary>

// エラー時のフォールバックUI
const UserListErrorFallback: React.FC<{
  error: Error;
  resetErrorBoundary: () => void;
}> = ({ error, resetErrorBoundary }) => {
  const reloadUserList = useSetAtom(userListAtom);
  const reset = () => {
    startTransition(() => {
      // Error Boundaryをリセット
      resetErrorBoundary();
      // atomの再実行をトリガー
      reloadUserList();
    });
  };
  return (
    <div>
      <p>エラーが発生しました: {error.message}</p>
      <button onClick={reset}>再試行</button>
    </div>
  );
};
```

ポイントは、`UserListErrorFallback`の中の`reset`関数です。ここで2つの処理を行っています。Error Boundaryのリセットと、atomの再実行のトリガーです。

鋭い方は、「`reloadUserList`を先に呼び出さないと`resetErrorBoundary`を呼び出した瞬間に`UserList`コンポーネントが再レンダリングされてまたエラーになってしまうのでは？」と思われるかもしれません。

しかし、そこは問題ありません。なぜなら、React 18以降ではステート更新のバッチ処理が実装されており、このように連続してステート更新を行った場合は両方の更新がまとめて処理されるからです。したがって、`resetErrorBoundary`によって再レンダリングされたタイミングではすでに`userListAtom`の再実行がトリガーされているため、エラーにはならずにローディング中のUIが表示されます。

## まとめ

この章では、Reactのエラーハンドリングのやり方について、特にSuspenseと絡めて解説しました。

従来のReactではあまりError Boundaryがあまり使われてこなかった印象があります。それは、非同期処理のエラーハンドリングが従来はError Boundaryの対応外だったからです。

Suspenseの登場により、非同期処理のエラーハンドリングもError Boundaryで行われるようになりました。それに合わせて、Error Boundaryをアプリに取り入れ、エラーUIの設計もきちんと行うことが重要になっています。

## 練習問題

この章で学んだError Boundaryとリトライの実装を組み合わせて、エラー時に再試行できるUIを作成してみましょう。

以下のアプリケーションは、ニュース記事一覧を取得して表示するものです。しかし、APIが50%の確率で失敗するにもかかわらず、エラーハンドリングが実装されていません。そのため、エラーが発生すると画面が真っ白になってしまいます。

```tsx
import { Suspense } from "react";
import { atom, useAtomValue } from "jotai";

type Article = { id: number; title: string };

// 50%の確率で失敗するAPI
async function fetchArticles(): Promise<Article[]> {
  await new Promise((resolve) => setTimeout(resolve, 500));
  if (Math.random() < 0.5) {
    throw new Error("記事の取得に失敗しました");
  }
  return [
    { id: 1, title: "Reactの最新アップデート" },
    { id: 2, title: "TypeScript 5.0の新機能" },
    { id: 3, title: "Next.js入門ガイド" },
  ];
}

const articlesAtom = atom(async () => {
  return fetchArticles();
});

// 記事一覧を表示するコンポーネント
const ArticleList: React.FC = () => {
  const articles = useAtomValue(articlesAtom);
  return (
    <ul>
      {articles.map((article) => (
        <li key={article.id}>{article.title}</li>
      ))}
    </ul>
  );
};

const App: React.FC = () => {
  return (
    <div>
      <h1>ニュース記事一覧</h1>
      <Suspense fallback={<p>読み込み中...</p>}>
        <ArticleList />
      </Suspense>
    </div>
  );
};
```

このアプリケーションを改良して、以下の要件を満たすようにしてください。

1. エラー発生時に、エラーメッセージと「再試行」ボタンを表示する
2. 「再試行」ボタンを押すと、記事の再取得を試みる

:::details 答え

```tsx
import { Suspense, startTransition } from "react";
import { useAtomValue, useSetAtom } from "jotai";
import { ErrorBoundary, type FallbackProps } from "react-error-boundary";

type Article = { id: number; title: string };

async function fetchArticles(): Promise<Article[]> {
  await new Promise((resolve) => setTimeout(resolve, 500));
  if (Math.random() < 0.5) {
    throw new Error("記事の取得に失敗しました");
  }
  return [
    { id: 1, title: "Reactの最新アップデート" },
    { id: 2, title: "TypeScript 5.0の新機能" },
    { id: 3, title: "Next.js入門ガイド" },
  ];
}

const articlesAtom = createReloadableAtom(async () => {
  return fetchArticles();
});

const ArticleList: React.FC = () => {
  const articles = useAtomValue(articlesAtom);
  return (
    <ul>
      {articles.map((article) => (
        <li key={article.id}>{article.title}</li>
      ))}
    </ul>
  );
};

// エラー時のフォールバックUI
const ArticleListFallback: React.FC<FallbackProps> = ({
  error,
  resetErrorBoundary,
}) => {
  const reloadArticles = useSetAtom(articlesAtom);

  const handleRetry = () => {
    startTransition(() => {
      resetErrorBoundary();
      reloadArticles();
    });
  };

  return (
    <div>
      <p>エラー: {error.message}</p>
      <button type="button" onClick={handleRetry}>
        再試行
      </button>
    </div>
  );
};

const App: React.FC = () => {
  return (
    <div>
      <h1>ニュース記事一覧</h1>
      <ErrorBoundary FallbackComponent={ArticleListFallback}>
        <Suspense fallback={<p>読み込み中...</p>}>
          <ArticleList />
        </Suspense>
      </ErrorBoundary>
    </div>
  );
};
```

ポイントは以下のとおりです。

1. **`createReloadableAtom`の使用**: 再試行機能を実装するには、atomの再読み込みをトリガーできる必要があります。以前の章で作成した`createReloadableAtom`を使うことで、書き込み操作で再読み込みをトリガーできるようになります。

2. **Error Boundaryの追加**: `ErrorBoundary`を`Suspense`の外側に配置し、`FallbackComponent`にフォールバックコンポーネントを渡しています。`FallbackProps`型を使うことで、`error`と`resetErrorBoundary`を受け取れます。

3. **`useSetAtom`で再読み込みをトリガー**: `articlesAtom`は`createReloadableAtom`で作られているため、書き込み操作（`reloadArticles()`）で再読み込みがトリガーされます。

ちなみに、この回答では`<h1>`タグをError Boundaryの外に配置しています。これにより、エラーが発生してもタイトルは表示されたままになり、ユーザーが何のページにいるのかを把握できます。Error Boundaryの配置を考える際には、このようにエラー時でも残しておきたい要素を意識するとよいでしょう。

:::
