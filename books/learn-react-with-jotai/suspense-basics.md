---
title: "Suspenseの基本"
---

この章では、Suspenseを学ぶ最初の一歩として、まずSuspenseの概念を解説します。

Suspenseは、**非同期処理によるデータ取得**をReactアプリケーションに組み込むための機能です。より正確に言えば、非同期処理を、Reactの基礎である**宣言的UI**にうまく組み込むために設計された機能なのです。

非同期処理は、「読み込み開始時の処理」「読み込み完了時の処理」「読み込み失敗時の処理」など、イベント駆動的な取り扱いが多く必要でした。これをReactアプリに組み込む際に、ともすれば手続き的なコードになってしまいます。

この問題に立ち向かい、宣言的UIであること、特に「UIはステートから計算されるものである」という原則を徹底したまま、非同期処理を取り扱うことに成功したのがSuspenseだと言えます。ピンと来ないかもしれませんが、心配する必要はありません。これにピンと来てもらうためにこの本を書いたのです。

## Suspenseの特徴

Suspenseはさまざまなことを考慮して設計された機能なので、その特徴もさまざまな側面から説明できます。

端的に分かりやすいところから説明すると、「**`isLoading`の管理が不要**」であることが挙げられます。次のようなステートは、Suspenseを使っているなら不要になります。

```ts
// ローディング中はtrue
const [isLoading, setIsLoading] = useState(false);
```

このような「読み込み中状態の管理」はReactがやってくれるので、アプリケーション上でステートを管理する必要はもうありません。

また、他のSuspenseの特徴として**Promiseとの統合**も挙げられます。PromiseはJavascriptの非同期処理に出てくるオブジェクトですが、Suspense以後のReactは、Promiseの処理をReactに任せることができます。React 19以降だれば`use` APIを使って`use(promise)`でPromiseの中身を取り出すという魔法のようなAPIを使えます。

```tsx
const UserProfile: React.FC<{ userPromise: Promise<User> }> = ({ userPromise }) => {
  // useでPromiseの中身を取り出す
  const user: User = use(userProfile);

  return <section>
    <h1>{user.name}のプロフィール</h1>
    ...
  </section>;
};
```

JavaScriptでは本来同期的にPromiseの中身を取得する方法は存在しないので、この`use` APIはSuspenseと連動し、裏で複数回Reactのレンダリングサイクルを回して“魔法”を実現しています。これについては後々解説します。

従来は、Promiseがあったら自分たちで非同期処理を取り扱わないといけませんでした。例えば`useEffect`の中でfetchしてPromiseに対して`.then`を呼ぶとかですね。

Suspenseの世界ではそれが変化し、場合によっては上のコード例のように「propsでPromiseを渡す」とか、従来やらなかったインターフェースを実践する余地が出てきたわけです。

この意味で、ReactでのPromiseの取り扱い方が変わったと言えます。

## Suspenseの動作

では、ここからSuspenseの具体的な動作について見ていきます。この本で完結するように解説しますが、さらにじっくり取り組みたい方はこちらの本もあわせて読むとよいでしょう。React 18が出る前に書いた本なので中身が古いですが、エッセンスは掴めるはずです。

https://zenn.dev/uhyo/books/react-concurrent-handson

この本では、執筆時点の最新バージョンであるReact 19を前提とします。このバージョンでSuspenseを使う際は`use` APIを用いることが推奨されています。

### 簡単な例

非同期処理によるデータ取得ということで、今回はこんな感じの非同期関数でデータが取得できることにします。

```ts
async function fetchUser(): Promise<User> {
  // 実際にはfetchとかでデータを取得してくる
  await new Promise((resolve) => setTimeout(resolve, 500));
  return { name: "ユーザー" };
}
```

とりあえず、雑だけど簡単な例をお見せします。

```tsx
const App: React.FC = () => {
  const userPromise: Promise<User> = useMemo(() => fetchUser(), []);

  return (
    <Suspense fallback={<p>Loading...</p>}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
};

const UserProfile: React.FC<{ userPromise: Promise<User> }> = ({ userPromise }) => {
  const user: User = use(userPromise);

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      ...
    </section>
  );
};
```

この例では、`App`がユーザーのデータ取得を担当し、`UserProfile`がユーザーデータの表示を担当しています。おkの例がどのように動作するか説明します。

まず、`UserProfile`は初回レンダリング時にPromiseに対して`use`を呼び出します。このとき、`UserProfile`は**サスペンド**します。

サスペンドとは、コンポーネントがまだPromiseを待っているためレンダリング完了できない状態を指します。このとき、`UserProfile`の実行は`use`で中断されます（内部的にはthrowで実行を中断しています）。

`UserProfile`がサスペンドしてしまったため、レンダリングができません。これに対処するのが`Suspense`コンポーネントです。

`Suspense`コンポーネントは、内部（children）でサスペンドが発生した際に、代わりに`fallback`を表示する機能を持ちます。

これにより、`App`をレンダリングした際、まずレンダリング結果は`<p>Loading...</p>`となり、これが画面に反映されます。

サスペンドしたコンポーネントは、待っていたPromiseが成功 (fulfill) したら自動的に**再レンダリング**が試みられます。つまり、`Suspense`の中身が再度レンダリングされます。

再レンダリングの際、同じように`use(userPromise)`を実行するわけですが、今回は`use`を実行してもサスペンドしません。なぜなら、`userPromise`はもう成功済であるということをReactが認識して、再レンダリング時に`use`がそれを返すからです。こうして、変数`user`には`userPromise`の結果が入り、レンダリングが続行します。

これにより今回は、最終的なレンダリング結果が`<section><h1>ユーザーさんのプロフィール</h1>...</section>`のようになります。

このように、コード上は「`use`を呼び出すとPromiseの中身が得られる」ように見える魔法のようなコードは、実際は1回レンダリングを中断（サスペンド）してあとで再開するというReactの働きによって実現していたのです。これがSuspenseの動作です。

### コンポーネントの中でfetchしたらだめなの？

上のコードを見ると、`App`がfetchを行って、`UserProfile`が`use`するという分担が見て取れます。このような分担が本当に必要なのか、疑問に思われたのではないでしょうか。特に、`UserProfile`がユーザーのプロフィールを表示する責務を持っているのであれば、`App`の中ではなく`UserProfile`の中でfetchも行うのが筋がいいように思えます。

まあ、結論としてはそれができないから上のようなサンプルコードになっているのですが、何がだめなのか見てみましょう。

```tsx:これだとだめなの？
const App: React.FC = () => {
  return (
    <Suspense fallback={<p>Loading...</p>}>
      <UserProfile  />
    </Suspense>
  );
};

const UserProfile: React.FC = () => {
  const userPromise: Promise<User> = useMemo(() => fetchUser(), []);

  const user: User = use(userPromise);

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      ...
    </section>
  );
};
```

これを実行して試してみましょう。

:::message
手元で試してみるためには、お手元のAIエージェントに上のコードをコピペで与えて「このコードを動かしたいので最低限のViteプロジェクトをセットアップして」と頼みましょう。
:::

試してみると、案の定うまく動きません。以下のような挙動になります。

- 画面はずっと「Loading...」のまま
- `fetchUser`が繰り返し呼び出され続けている

#### コンポーネントの中でfetchしてもうまく動かない理由

こうなる理由は、この場合 **`use`に毎回新しいPromiseが渡されるから**です。そのため、「Promiseを待つ→再レンダリング→次のPromiseが与えられる→待つ→……」という無限ループになっています。

この場合の動きを詳しく見ていきましょう。

`UserProfile`が最初にレンダリングされると、`userPromise`に`fetchUser`の結果（新しいPromise）が入り、それを`use`したことでサスペンドします。このときのPromiseを「Promise1」としましょう。

Promise1が成功した場合は再レンダリングが発生します。このとき、`useMemo`を使っているにもかかわらず、**`fetchUser`が再度実行されます**。その結果（Promise2）が`userPromise`に入ります。

再度`fetchUser`を実行したので、その結果を再度待つ必要があり、再サスペンドします。そしてサスペンドが明けたらまた`fetchUser`を実行……という流れです。

#### useMemoが効いていない？

では、なぜ`useMemo`が効かずに再度`fetchUser`が実行されてしまうのでしょうか。その理由は、**`UserProfile`が1回もレンダリング成功していないから**です。

`UserProfile`がサスペンドした場合、`<Suspense>`の中身ではなく`fallback`がレンダリングされますよね。その時点で、`UserProfile`は**レンダリングされなかったことになります**。

`useMemo`の役割は「**前回のレンダリング**で計算した値を再利用すること」ですから、サスペンド明けのレンダリングでは「前回のレンダリング」が存在していません。そのため、`useMemo`があっても、`UserProfile`がレンダリングされるたびに再度`fetchUser`が実行されていたのです。

なので、サスペンド明けにレンダリングがもう一度行われるのは、「再レンダリング」というよりは「レンダリングに**再挑戦**」といったほうがより的確かもしれません。初回レンダリングが中断された（サスペンド）ので、Promiseが成功してからもう一度初回レンダリングを試行してみるということです。

別の言い方をすると、`useMemo`のようなフックはコンポーネントの**記憶領域**にデータを保存します。この記憶領域は、コンポーネントのレンダリング成功を以て確保され、保存されるものです。初回レンダリングが中断された場合はこの記憶領域も確保されません。そのため、再挑戦時は記憶領域が無い状態で再びレンダリングされることになり、前回の`useMemo`の結果を再利用できないのです。

### Suspenseを使うために必要なこと

以上のことから分かるように、Suspenseを使う場合に必要なのは、**コンポーネントの外にPromiseを持っておくこと**です。`use`の仕組み上、サスペンド前後の2回のレンダリングで同じPromiseを`use`に与える必要があります。`UserProfile`コンポーネントの中のuseStateやuseMemoでは、これは不可能です。

そのため、Promiseを外に持っておく必要があります。最初の例では、Promiseの保管は`UserProfile`の外の`App`が担当していたため、`UserProfile`の記憶領域が破棄されても問題なくPromiseを保管することができていました。

次以降の章では、この問題に対してステート管理ライブラリ（jotai）を用いて立ち向かうことになります。

## Suspenseは「Suspense境界」を定める

Suspenseの基本的な動作は以上で分かりましたが、ここでひとつ重要な考え方をお伝えしておきます。

それは、Suspenseはその内側と外側の間の**境界**として働くものだということです。これはSuspense境界 (Suspense boundary) と呼ばれます。

この境界は**サスペンドの影響範囲**を区切るものです。境界の内側で発生したサスペンドは、Suspenseによってキャッチされ処理されるため、境界の外側には影響しません。これはtry-catchに似ていますね。try-catchも、その内側で発生した例外をキャッチして、外に例外が波及しないようにする機能です。

逆に、Suspenseの中でサスペンドが発生した場合、同じSuspenseに属するコンテンツはまとめて`fallback`に置き換えられます。つまり、Suspenseの中のコンテンツは一蓮托生だということです。特に、Suspenseの中で複数のサスペンドが発生した場合、それらが全て完了しないとSuspenseの中身は表示されません。

このため、Suspenseを活用する場合、アプリ中のさまざまな場所でサスペンドが起きることは前提として、それらの影響範囲を設計し、Suspense境界を適切に引くことが重要になります。

例えば、ページの一部に特に読み込みに時間がかかる部分がある場合は、そこだけ独自のSuspense境界の中に置くのが有効です。そうすることで、そこだけサスペンドが長引いてもページの他の部分は表示させることができます。

## まとめ

この章では、Suspenseの基本的な挙動を解説しました。特に、サスペンドするコンポーネントがそのためのデータを自身のステート内に持つことはできないというのは重要な点です。

次章では、Suspenseとjotaiの組み合わせについて解説します。