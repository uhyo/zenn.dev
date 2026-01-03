---
title: "トランジションとUIバージョニング"
---

さらにトランジションの話が続きます。トランジションを制する者がSuspenseを制するのです。

この章では、トランジションを活用するに当たって、5章で説明した「**UIバージョニング**」の概念が重要になることを説明します。

## 今回のサンプル: ユーザーIDと再読み込み

復習すると、5章では派生atomの再読み込みのためにkeyを使う方法を学んだのでしたね。

そこで、この章は5章のサンプルをアレンジしたものを題材とします。

まず、再読み込みによる変化が分かるように`name`だけでなく`followers`を追加し、`followers`は読み込みのたびに変わるようにしました。また、練習問題の答えも採り入れています。

実際に実行するとユーザープロフィールが表示され、更新ボタンで再読み込みを行うことができます。

さらに、`useEffect`と`setInterval`を使って1秒ごとに自動で再読み込みを行うようにしちゃいました。

```tsx
interface User {
  name: string;
  followers: number;
}

async function fetchUser(): Promise<User> {
  await new Promise((resolve) => setTimeout(resolve, 500));
  return {
    name: "ユーザー",
    followers: Math.floor(Math.random() * 10000),
  };
}

function createReloadableAtom<T>(
  getter: (get: Getter) => T
) {
  const refetchKeyAtom = atom(0);

  return atom(
    (get) => {
      get(refetchKeyAtom);
      return getter(get);
    },
    (get, set) => {
      set(refetchKeyAtom, (key) => key + 1);
    }
  );
}

// 使用例
const userAtom = createReloadableAtom(async () => {
  const user = await fetchUser();
  return user;
});

const UserProfile: React.FC = () => {
  const user = useAtomValue(userAtom);
  const reloadUser = useSetAtom(userAtom);

  useEffect(() => {
    const intervalId = setInterval(() => {
      reloadUser();
    }, 1000);
    return () => clearInterval(intervalId);
  }, [reloadUser]);

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      <p>フォロワー数: {user.followers}</p>
      <button type="button" onClick={() => reloadUser()}>再読み込み</button>
    </section>
  );
};

const App: React.FC = () => {
  return (
    <Suspense fallback={<p>Loading...</p>}>
      <UserProfile />
    </Suspense>
  );
};
```

これを実行すると、頻繁にサスペンドして「Loading...」に戻るせわしないUIになりました。

### トランジションを足す

では、これにトランジションを足してみましょう。再読み込みをトランジションでラップします。

```tsx
const UserProfile: React.FC = () => {
  const user = useAtomValue(userAtom);
  const reloadUser = useSetAtom(userAtom);
  const [isPending, startTransition] = useTransition();

  useEffect(() => {
    const intervalId = setInterval(() => {
      startTransition(() => {
        reloadUser();
      });
    }, 1000);
    return () => clearInterval(intervalId);
  }, [reloadUser, startTransition]);

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      <p>フォロワー数: {user.followers}</p>
      <button
        type="button"
        onClick={() => {
          startTransition(() => {
            reloadUser();
          });
        }}
      >
        再読み込み
        {isPending && "（更新中...）"}
      </button>
    </section>
  );
};
```

まだちょっとせわしないですが、トランジションの効果でユーザーのプロフィールが画面から消えてしまうことは無くなりました。

### ユーザーIDの切り替え機能を足す

まだ機能を足します。ユーザーIDを切り替える機能を足しましょう。以下のように、`userIdAtom`というプリミティブatomを用意し、そこに取得すべきユーザーIDを保持します。

```tsx
const userIdAtom = atom("user1");

const userAtom = createReloadableAtom(async (get) => {
  const userId = get(userIdAtom);
  const user = await fetchUser(userId);
  return user;
});

// fetchUserもuserIdを受け取るように変更
async function fetchUser(userId: string): Promise<User> {
  await new Promise((resolve) => setTimeout(resolve, 500));
  return {
    name: `ユーザー(${userId})`,
    followers: Math.floor(Math.random() * 10000),
  };
}
```

もちろんユーザーIDを切り替えるUIも必要ですね。以下のように`UserSelector`コンポーネントを追加します。

```tsx
const UserSelector: React.FC = () => {
  const setUserId = useSetAtom(userIdAtom);

  const handleSelectUser = (id: string) => {
    startTransition(() => {
      setUserId(id);
    });
  };

  return (
    <div>
      <button type="button" onClick={() => handleSelectUser("user1")}>ユーザー1を選択</button>
      <button type="button" onClick={() => handleSelectUser("user2")}>ユーザー2を選択</button>
    </div>
  );
};
```

これを`App`コンポーネントに組み込みます。

```tsx
const App: React.FC = () => {
  return (
    <>
      <UserSelector />
      <Suspense fallback={<p>Loading...</p>}>
        <UserProfile />
      </Suspense>
    </>
  );
};
```