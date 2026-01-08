---
title: "AbortSignalを使って非同期処理の中断に対応する"
---

非同期処理を取り扱う際に忘れてはならないのが、**AbortSignal**への対応です。AbortSignalは、非同期処理を中断するための仕組みを提供します。非同期処理は時間がかかるため、**非同期処理の途中で結果がもう不要になる場合**があります。その場合、無駄な処理を続けるのではなく中断できると効率的です。

前章の例で言うと、例えば「`userId === "user1"`のデータの取得が完了する前にユーザーがボタンを押して`userId === "user2"`に切り替えた」場合が該当します。この場合、`user1`のデータが取得完了しても、それはもう使われません。ステートが`user2`に変わった時点でReactは`user2`のUIの描画を開始しているため、`user1`のデータが取得完了してもUIには反映されないのです。

そこで、この章ではAbortSignalを使って非同期処理の中断に対応する方法を解説します。

ただし、AbortSignalはReactやjotaiとは直接関係ない、JavaScript（正確に言えばDOM）の機能です。したがって、この本では詳細な解説はせず、jotaiでの利用方法に絞って解説します。

## AbortSignalとは

**AbortSignal**は、その名のとおり「中断 (abort)」を「伝える (signal)」ための仕組みです。「AbortSignalを作った人」が、「AbortSignalを受け取った人」に対して「中断してね」と伝えるために使います。

そのため、`fetch`など非同期処理を行うWeb APIの多くは、AbortSignalを受け取ることができます。AbortSignalを受け取った非同期処理は、AbortSignalから「中断してね」と伝えられた場合に中断します。

## jotaiでAbortSignalを使う方法

jotaiでは、実は**派生atomの読み取り関数に、引数の一部としてAbortSignalが渡される**ようになっています。この場合、AbortSignalを作った人はjotaiのランタイムであり、AbortSignalを受け取った人は派生atomの読み取り関数です。

このAbortSignalは、読み取り関数が非同期処理を行う場合に有効で、**その読み取り処理の中断**を伝えるために使われます。読み取りの中断は、当たり前ですが、もう読み取りが不要になった場合に発生します。例えば、派生atomの依存先が変化し、再計算が必要になった場合などです。

基本的なユースケースでは、読み取り関数は、受け取ったAbortSignalをそのまま`fetch`などの非同期処理に渡せばOKです。

以下が具体的な例です。

```ts
const userAtom = atom(async (get, { signal }): Promise<User> => {
  const userId = get(userIdAtom);
  const response = await fetch(`/api/users/${userId}`, { signal });
  const user: User = await response.json();
  return user;
});
```

この例では、`userAtom`の読み取り関数が`signal`を受け取っています。そして、その`signal`を`fetch`に渡しています。これにより、`fetch`中に`userAtom`の読み取りが中断された場合、AbortSignalの働きにより`fetch`も中断されます。

中断された場合、`fetch`は例外を発生させます。典型的には、この場合は`AbortError`という名前の`DOMException`が発生します。つまり、`response`が得られることはなく、それ以降のコードも実行されません。

この帰結として、`userAtom`の結果もエラーということになりますが、中断の場合にはエラー処理はそこまで気にする必要はありません。それは、以下の2つの理由からです。

- もう中断された読み取りなので、エラーがjotaiにキャッシュされることはない。
- 中断された読み取りの結果を使うこともないので、エラーがUIに影響を与えることもない。

今回はエラーのことは気にしなくても問題ありません。一般のエラー処理に関しては別の章で解説します。

## 動作確認してみる

前章の例では`fetch`ではなくこんな疑似的な実装にしていました。これをAbortSignalに対応させます。

```ts
async function fetchUser(userId: string): Promise<User> {
  // 実際にはfetchとかでデータを取得してくる
  await new Promise((resolve) => setTimeout(resolve, 500));
  return { name: `ユーザー${userId}` };
}
```

Reactとは関係ありませんが、興味がある方は**練習問題**として自力でやってみてください。

:::details 答え

```ts
async function fetchUser(userId: string, signal: AbortSignal): Promise<User> {
  // 実際にはfetchとかでデータを取得してくる
  await new Promise((resolve, reject) => {
    signal.throwIfAborted();
    setTimeout(resolve, 500);
    signal.addEventListener("abort", () => {
      reject(signal.reason);
    }, { once: true });
  });
  return { name: `ユーザー${userId}` };
}
```

この例では、受け取ったAbortSignalに`addEventListener`で`abort`イベントのリスナーを登録しています。`abort`イベントはAbortSignalが中断を伝えるときに発生します。その場合は`reject`を呼び出し、Promiseを失敗させます。

ちなみに、`new Promise`の時に得られる`resolve`や`reject`は、複数回呼び出した場合は最初の1回だけが有効になります。つまり、このコードでは`setTimeout`による`resolve`と`abort`イベントによる`reject`を競争させているのです。
:::

前章のサンプル（`userId`をatomに入れるほう）を直す形で`userAtom`を修正すると、このようになります。

```ts
const userIdAtom = atom<string | null>(null);

const userAtom = atom(async (get, { signal }): Promise<User | null> => {
  const userId = get(userIdAtom);
  // ユーザーIDが設定されていない場合はuserAtomもnullを返す
  if (userId === null) return null;

  const user = await fetchUser(userId, signal);
  return user;
});
```

これも先ほどの説明と同様に、受け取った`signal`を`fetchUser`に渡しているだけです。

基本的に、AbortSignalを受け取るということは、中断に適切に対応する責任を負うということです。jotaiはこのように読み取り関数には`signal`を渡してきますので、`signal`を無視するのはよろしくありません。原則として、非同期処理を行う場合はきちんと`signal`に対応しましょう。

:::message
ただし、この本ではサンプルコードを簡潔にするため、この章以外では`signal`の対応は省略しています。ご了承ください。
:::

実際に動作確認してみましょう。方法は、user1に切り替えてから0.5秒以内にuser2に切り替えます（間に合わない場合はコードを編集して`fetchUser`の待ち時間を長くしましょう）。

そうした場合、`userAtom`の値が、非同期処理が完了しないうちに再計算されることになります。このとき、最初の非同期処理は中断され、2つ目の非同期処理だけが完了します。結果として、UIには正しくuser2のデータだけが表示されるはずです。

user1→user2→user1→……という切り替えを素早く繰り返してみると、再読み込みをしつづけるのでずっとサスペンド中のままになることも観察できます。

これだとAbortSignalがきちんと働いているか分かりにくいので、試しに`abort`イベント発生時に`console.log`でログを出してみましょう。確かに非同期処理が中断されていることが分かるはずです。

また、この場合とくにコンソールにエラー等は出力されないはずです。非同期処理の中断は前述のとおりエラー（AbortError）として扱われますが、中断の場合はjotaiがうまく処理してくれるため、そのエラーが外に漏れることはありません。

## atomファクトリーの場合

前章では、パラメータ付きの非同期処理のやり方として、派生atomの依存先にパラメータを保持する方法（この章で扱った方法）以外にも、atomファクトリーを使う方法があると説明しました。

そちらの場合も、もちろんこのようにAbortSignalに対応することができます。

```ts
const userAtomFamily = atomFamily((userId: string) =>
  atom(async (get, { signal }): Promise<User> => {
    const user = await fetchUser(userId, signal);
    return user;
  })
);
```

実は、この場合は**user1からuser2に切り替えても非同期処理は中断されません**。

なぜなら、atomファクトリーの場合、`userAtomFamily("user1")`と`userAtomFamily("user2")`は**別々のatom**だからです。したがって、`userId`を`user1`から`user2`に変えた場合、user1のatomは参照されなくなりますが、jotaiのランタイムはそれを中断しません。単に参照されなくなったatomとして放置されるだけです。この場合、結果はjotaiによりキャッシュされ、あとから参照された場合に利用されます。

要するに、jotaiによる非同期処理の中断が行われるのは、**同じatomの再計算が必要になった場合**に限られるのです。

## まとめ

この章では、AbortSignalを使って非同期処理の中断に対応する方法を解説しました。jotaiでは派生atomの読み取り関数にAbortSignalが渡されるため、それを非同期処理に渡すだけで対応できます。

AbortSignalを渡された以上は、特別な理由が無い限りはきちんと対応するようにしましょう。

## 練習問題

複数の非同期処理を同時に行う場合のAbortSignal対応を練習しましょう。

以下のような状況を考えます。ユーザー情報と、そのユーザーの投稿一覧を同時に取得する派生atomを作ります。

```ts
type User = { name: string };
type Post = { id: number; title: string };

async function fetchUser(userId: string, signal: AbortSignal): Promise<User> {
  // 省略: signalに対応したfetch
}

async function fetchPosts(userId: string, signal: AbortSignal): Promise<Post[]> {
  // 省略: signalに対応したfetch
}

const userIdAtom = atom<string>("user1");

// このatomを完成させてください
const userDataAtom = atom(async (get, { signal }): Promise<{ user: User; posts: Post[] }> => {
  const userId = get(userIdAtom);
  // userとpostsを同時に取得して返す
  // ヒント: Promise.allを使う
  ???
});
```

`fetchUser`と`fetchPosts`を**同時に**実行し、両方の結果を返すようにしてください。両方の非同期処理にAbortSignalを正しく渡すことがポイントです。

:::details 答え

```ts
const userDataAtom = atom(async (get, { signal }): Promise<{ user: User; posts: Post[] }> => {
  const userId = get(userIdAtom);
  const [user, posts] = await Promise.all([
    fetchUser(userId, signal),
    fetchPosts(userId, signal),
  ]);
  return { user, posts };
});
```

`Promise.all`を使って`fetchUser`と`fetchPosts`を並列に実行しています。重要なのは、**両方の関数に同じ`signal`を渡している**ことです。

これにより、`userDataAtom`の読み取りが中断された場合、両方のfetch処理が同時に中断されます。もし片方にしか`signal`を渡していなかったら、もう片方の処理は中断されずに無駄に実行され続けてしまいますね。

このように、複数の非同期処理を組み合わせる場合は、**すべての非同期処理に同じAbortSignalを渡す**ことで、中断が発生したときに確実にすべての処理を止めることができます。
:::
