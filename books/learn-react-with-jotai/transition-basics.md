---
title: "トランジションの基本"
---

この章では、**トランジション**について解説します。トランジションはReactの機能のひとつで、Suspenseと深い関係があります。Suspenseをしっかりと活用したアプリケーションを作るためには、トランジションの理解が欠かせません。

## トランジションとは

Reactでは、**ステート更新をトランジションにする**ことができます。意味付けとしては、トランジションとなったステート更新は「**優先度が低い更新**」として扱われます。

分かりやすい違いは、**そのステート更新によってサスペンドが発生した場合の挙動**です。これまでのSuspenseの説明では、`Suspense`コンポーネントの中でサスペンドが発生すると、その`Suspense`の中身（`children`）は片付けられて、代わりに`fallback`が表示されると説明してきました。

しかし、トランジションによるステート更新でサスペンドが発生した場合は、`Suspense`の中身は**片付けられずにそのまま残されます**。つまり、**ステート更新前の古いUIがそのまま表示され続ける**のです。そして、非同期処理が完了してサスペンドが解消されたときに、新しいUIに切り替わります。

つまり、「古いUI→フォールバック→新しいUI」という流れではなく、古いUIの表示時間を延長して、「古いUI→新しいUI」という流れになるわけです。

特にフォールバックUIが一瞬だけ表示されてしまうと、ユーザーには画面がちらついているように見えます。トランジションを使うことで、こうしたちらつきを防止できます。

## トランジションの使い方

トランジションを使うには、Reactから提供されている`useTransition`フックか、`startTransition`関数を使います。まずは`startTransition`関数を使う方法を紹介します。

ステート更新によってサスペンドを発生させる必要があるので、これまでの章から例を引っ張ってきましょう。まずはトランジション導入前の状態を示します。

```tsx
interface User {
  name: string;
}

async function fetchUser(userId: string): Promise<User> {
  await new Promise((resolve) => setTimeout(resolve, 500));
  return  { name: `ユーザー${userId}` };
}

const userAtomFamily = atomFamily((userId: string) =>
  atom(async (): Promise<User> => {
    const user = await fetchUser(userId);
    return user;
  })
);

const App: React.FC = () => {
  const [userId, setUserId] = useState("user1");
  const handleSelectUser = (id: string) => {
    setUserId(id);
  }
  return (
    <>
      <UserSelector onSelectUser={handleSelectUser} />
      <Suspense fallback={<p>Loading...</p>}>
        <UserProfile userId={userId} />
      </Suspense>
    </>
  );
};

const UserProfile: React.FC<{ userId: string }> = ({ userId }) => {
  const user: User = useAtomValue(userAtomFamily(userId));

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      ...
    </section>
  );
};

const UserSelector: React.FC<{ onSelectUser: (id: string) => void }> = ({ onSelectUser }) => {
  return (
    <div>
      <button type="button" onClick={() => onSelectUser("user1")}>ユーザー1を選択</button>
      <button type="button" onClick={() => onSelectUser("user2")}>ユーザー2を選択</button>
    </div>
  );
};
```

このアプリケーションでは最初`userId`ステートが`"user1"`に設定されており、`UserProfile`コンポーネントがレンダリングされると`userAtomFamily("user1")`が読み取られます。するとサスペンドが発生し、0.5秒後にユーザー1のデータが取得されて表示されます。

ボタンを押して`userId`ステートを`"user2"`に変更すると、`UserProfile`コンポーネントが再レンダリングされ、`userAtomFamily("user2")`が読み取られます。このとき`UserProfile`コンポーネントは再度サスペンドし、再びフォールバックUIが表示されます。0.5秒後にユーザー2のデータが取得されて表示されます。

### startTransitionを使う

では、`userId`ステートの更新をトランジションにしてみましょう。`handleSelectUser`関数を以下のように書き換えます。

```tsx
const handleSelectUser = (id: string) => {
  // import { startTransition } from 'react'; でインポートしておく
  startTransition(() => {
    setUserId(id);
  });
}
```

このように、`startTransition`はコールバック関数を受け取り、その関数を即時実行します。コールバック関数内で行われたステート更新はすべてトランジション扱いになります。

こうすると、`userId`が`"user2"`になったときの挙動が変わります。ボタンを押しても画面にはユーザー1のプロフィールがそのまま表示され続け、フォールバックUIには切り替わりません。そして0.5秒後にユーザー2のデータが取得されると、プロフィールがユーザー2のものに切り替わります。先ほど説明したとおり、フォールバックUIを挟まずに「古いUI→新しいUI」という流れになっています。

これで「一瞬フォールバックUIがちらつくのを防ぐ」という目的は達成できていますが、どうも別の問題があるように思えます。それは、**ボタンを押した瞬間のフィードバックがない**ことです。ユーザーはボタンを押したのに何も起こらないので、操作できていないのではないかと不安になるかもしれません。

これを解決するのが、`useTransition`フックです。

### useTransitionを使う

`useTransition`フックを使うと、トランジションの進行状態を知ることができます。以下のように書き換えます。

```tsx
const App: React.FC = () => {
  const [isPending, startTransition] = useTransition();
  const [userId, setUserId] = useState("user1");
  const handleSelectUser = (id: string) => {
    startTransition(() => {
      setUserId(id);
    });
  }
  return (
    <>
      <UserSelector onSelectUser={handleSelectUser} />
      <Suspense fallback={<p>Loading...</p>}>
        <UserProfile userId={userId} isPending={isPending} />
      </Suspense>
    </>
  );
};

const UserProfile: React.FC<{ userId: string; isPending: boolean }> = ({ userId, isPending }) => {
  const user: User = useAtomValue(userAtomFamily(userId));

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      <p>{isPending && <span> (読み込み中...)</span>}</p>
      ...
    </section>
  );
};
```

`useTransition`フックは2つの値を返します。1つ目はトランジションが進行中かどうかを示す`isPending`ブール値、2つ目はトランジションを開始するための`startTransition`関数です。

`startTransition`関数は、先ほどの、Reactから直接インポートして使う`startTransition`関数と同じ使い方ができます。それに加えて`isPending`が得られるのが`useTransition`フックの特徴です。

この`isPending`は通常は`false`ですが、トランジションによるステート更新が進行中でサスペンドが発生している間は`true`になります。これを使って、`UserProfile`コンポーネント内で「(読み込み中...)」のようなフィードバックを表示しています。

この場合、ボタンを押して`userId`が`"user2"`になると、以下の挙動になります。

1. ボタンを押すとすぐに`isPending`が`true`になり、「(読み込み中...)」が表示される
2. 画面にはユーザー1のプロフィールがそのまま表示され続ける
3. 0.5秒後にユーザー2のデータが取得され、プロフィールがユーザー2のものに切り替わると同時に`isPending`が`false`になり、「(読み込み中...)」が消える

表で整理するとこのようになります。

| 状況 | `userId` | `isPending` | 表示されるUI |
| --- | --- | --- | --- |
| 初期状態 | `"user1"` | false | ユーザー1のプロフィール |
| ボタンを押した直後 | `"user2"` | true | ユーザー1のプロフィール + (読み込み中...) |
| データ取得後 | `"user2"` | false | ユーザー2のプロフィール |

これにより、ユーザーはボタンを押したことに対するフィードバックを得られつつ、フォールバックUIのちらつきは防止できます。`isPending`は、フォールバックUIとは異なり、現在の表示に対する「プラスアルファ」の表示をするのに適しています。例えば、ローディングスピナーを表示したり、古い表示を薄く出したりするのも良いでしょう。

## 複数のトランジションを使い分ける

トランジションは、アプリケーションの中で複数使い分けることもできます。例えば、以下のように2つの`useTransition`フックを使うことも可能です。

```tsx
const App: React.FC = () => {
  const [isPendingA, startTransitionA] = useTransition();
  const [isPendingB, startTransitionB] = useTransition();
  ...
};
```

この場合、`startTransitionA`で開始したトランジションは`isPendingA`に影響し、`startTransitionB`で開始したトランジションは`isPendingB`に影響します。

トランジション中にどのようなフィードバックを表示すべきかは、場合によって異なるかもしれません。特に、UIのどこに影響を与えるかです。

トランジション中の典型的な表示は、もう古くなった情報を薄く表示したり、ローディングスピナーを表示したりすることです。そうなると、「そのトランジションによってUIのどこが古くなるのか」とか、「これからUIのどこに変化があるのか」に応じてトランジションを使い分けるのが自然ですね。

実践レベルのアプリケーションではトランジションの使い分けも考えて設計する必要があるでしょう。

## まとめ

この章では、トランジションの基本的な使い方を説明しました。トランジションを使うことで、SuspenseによるフォールバックUIのちらつきを防止しつつ、ユーザーに対して操作が受け付けられたことをフィードバックすることができます。

トランジションはSuspenseを活用したアプリケーションを作る上で欠かせない機能です。ぜひ積極的に活用してみてください。

## 練習問題

この章で学んだ「複数のトランジションを使い分ける」を実践してみましょう。

以下のアプリケーションは、ダッシュボードページです。「ユーザープロフィール」セクションと「投稿一覧」セクションがあり、それぞれユーザーやカテゴリを選択すると切り替わります。

```tsx
type User = { name: string };
type Post = { id: number; title: string };
type Category = "tech" | "life" | "news";

async function fetchUser(userId: string): Promise<User> {
  await new Promise((resolve) => setTimeout(resolve, 500));
  return { name: `ユーザー${userId}` };
}

async function fetchPostsByCategory(category: Category): Promise<Post[]> {
  await new Promise((resolve) => setTimeout(resolve, 800));
  const posts: Record<Category, Post[]> = {
    tech: [{ id: 1, title: "Reactの最新動向" }, { id: 2, title: "TypeScript入門" }],
    life: [{ id: 3, title: "おすすめの本" }, { id: 4, title: "休日の過ごし方" }],
    news: [{ id: 5, title: "新機能リリース" }],
  };
  return posts[category];
}

const userAtomFamily = atomFamily((userId: string) =>
  atom(async () => fetchUser(userId))
);

const postsByCategoryAtomFamily = atomFamily((category: Category) =>
  atom(async () => fetchPostsByCategory(category))
);

const Dashboard: React.FC = () => {
  const [userId, setUserId] = useState("user1");
  const [category, setCategory] = useState<Category>("tech");

  return (
    <div>
      <section>
        <h2>ユーザープロフィール</h2>
        <UserSelector onSelectUser={setUserId} />
        <Suspense fallback={<p>Loading user...</p>}>
          <UserProfile userId={userId} />
        </Suspense>
      </section>

      <section>
        <h2>投稿一覧</h2>
        <CategorySelector onSelectCategory={setCategory} />
        <Suspense fallback={<p>Loading posts...</p>}>
          <PostList category={category} />
        </Suspense>
      </section>
    </div>
  );
};

const UserProfile: React.FC<{ userId: string }> = ({ userId }) => {
  const user = useAtomValue(userAtomFamily(userId));
  return <p>{user.name}さんのプロフィール</p>;
};

const PostList: React.FC<{ category: Category }> = ({ category }) => {
  const posts = useAtomValue(postsByCategoryAtomFamily(category));
  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
};

const UserSelector: React.FC<{ onSelectUser: (id: string) => void }> = ({ onSelectUser }) => (
  <div>
    <button type="button" onClick={() => onSelectUser("user1")}>ユーザー1</button>
    <button type="button" onClick={() => onSelectUser("user2")}>ユーザー2</button>
  </div>
);

const CategorySelector: React.FC<{ onSelectCategory: (cat: Category) => void }> = ({ onSelectCategory }) => (
  <div>
    <button type="button" onClick={() => onSelectCategory("tech")}>Tech</button>
    <button type="button" onClick={() => onSelectCategory("life")}>Life</button>
    <button type="button" onClick={() => onSelectCategory("news")}>News</button>
  </div>
);
```

このアプリケーションでは、ユーザーやカテゴリを切り替えるたびにフォールバックUIが表示されてしまいます。これを改善するためにトランジションを導入しましょう。

ユーザープロフィールと投稿一覧は別々に読み込まれるため、それぞれ**独立したトランジション**を使います。つまり、ユーザーを切り替えたときは「ユーザープロフィール」セクションだけに「(読み込み中...)」が表示され、カテゴリを切り替えたときは「投稿一覧」セクションだけに「(読み込み中...)」が表示されるようにしてください。

:::details 答え

```tsx
const Dashboard: React.FC = () => {
  const [isPendingUser, startTransitionUser] = useTransition();
  const [isPendingPost, startTransitionPost] = useTransition();

  const [userId, setUserId] = useState("user1");
  const [category, setCategory] = useState<Category>("tech");

  const handleSelectUser = (id: string) => {
    startTransitionUser(() => {
      setUserId(id);
    });
  };

  const handleSelectCategory = (cat: Category) => {
    startTransitionPost(() => {
      setCategory(cat);
    });
  };

  return (
    <div>
      <section>
        <h2>ユーザープロフィール {isPendingUser && "(読み込み中...)"}</h2>
        <UserSelector onSelectUser={handleSelectUser} />
        <Suspense fallback={<p>Loading user...</p>}>
          <UserProfile userId={userId} />
        </Suspense>
      </section>

      <section>
        <h2>投稿一覧 {isPendingPost && "(読み込み中...)"}</h2>
        <CategorySelector onSelectCategory={handleSelectCategory} />
        <Suspense fallback={<p>Loading posts...</p>}>
          <PostList category={category} />
        </Suspense>
      </section>
    </div>
  );
};
```

ポイントは、`useTransition`を**2つ**使っていることです。

- `[isPendingUser, startTransitionUser]`はユーザープロフィールの切り替え専用
- `[isPendingPost, startTransitionPost]`は投稿一覧の切り替え専用

これにより、ユーザーを切り替えたときは`isPendingUser`だけが`true`になり、カテゴリを切り替えたときは`isPendingPost`だけが`true`になります。

もし1つの`useTransition`だけを使って両方のステート更新に使いまわしていたらどうなるでしょうか。その場合、どちらの操作をしても同じ`isPending`が`true`になってしまいます。ユーザーを切り替えただけなのに投稿一覧にも「(読み込み中...)」が表示されてしまい、ユーザーに誤った情報を与えてしまいます。

このように、**UIのどの部分が読み込み中なのか**を正確に伝えるためには、トランジションを適切に分離することが重要です。

:::
