---
title: "jotai-eagerを使う"
---

この章では、**jotai-eager**というライブラリを紹介します。

https://github.com/jotaijs/jotai-eager

このライブラリは、**無駄な非同期処理を避け、サスペンドを削減する**ために有用です。

jotaiの機能を活用して状態管理を行う場合、このライブラリを使ったほうがいい場面に遭遇することがあります。

## 今回の題材

この章では、前章までとは違うあらたな題材を用います。ちょっと応用的な例です。

今回は、ユーザー一覧からユーザーをチェックボックスで選択できることにしましょう。ただし、ユーザー一覧は非同期に取得されるものとします。

```tsx
interface User {
  id: string;
  name: string;
}

async function fetchUsers(): Promise<User[]> {
  await new Promise((resolve) => setTimeout(resolve, 500));
  return new Array(10).fill(null).map((_, i) => ({
    id: `user${i + 1}`,
    name: `ユーザー${i + 1}`,
  }));
}
```

この非同期関数を使って、ユーザー一覧を表示し、チェックボックスで選択できるようにします。

```tsx
const usersAtom = atom(async () => {
  const users = await fetchUsers();
  return users;
});

const selectedUserIdsAtom = atom<Set<string>>(new Set());

const toggleUserSelectionAtom = atom(
  null,
  (get, set, userId: string) => {
    const selectedUserIds = new Set(get(selectedUserIdsAtom));
    if (selectedUserIds.has(userId)) {
      selectedUserIds.delete(userId);
    } else {
      selectedUserIds.add(userId);
    }
    set(selectedUserIdsAtom, selectedUserIds);
  }
);

const UserList: React.FC = () => {
  const users = useAtomValue(usersAtom);
  const selectedUserIds = useAtomValue(selectedUserIdsAtom);
  const toggleUserSelection = useSetAtom(toggleUserSelectionAtom);

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>
          <label>
            <input
              type="checkbox"
              checked={selectedUserIds.has(user.id)}
              onChange={() => toggleUserSelection(user.id)}
            />
            {user.name}
          </label>
        </li>
      ))}
    </ul>
  );
};

const App: React.FC = () => {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <UserList />
    </Suspense>
  );
};
```

このコードをコピペすれば、以下の挙動をするアプリケーションが動くはずです。

- 最初に「Loading...」が表示され、500ミリ秒にユーザー一覧が表示される
- チェックボックスでユーザーを選択・解除できる

コードについてはこれまでの章で説明した内容の組み合わせで理解できます。チェックボックスの切り替えを書き込み専用atomで行っているのがjotai的ですね。

## 初期状態を非同期処理に依存させる

では、ここでちょっとしたお題です。

現在、初期状態は何もチェックされていません。これを仕様変更して、**初期状態で全てのユーザーが選択されているようにしたい**です。どうすればいいでしょうか？

プリミティブatomの初期状態は`atom()`の引数で決まるため、他のatomの状態に依存させることはできませんね。ひと工夫必要です。

一つの方法として、プリミティブatomと派生atomを組み合わせる方法があります。

```ts
const internalSelectedUserIdsAtom = atom<Set<string> | null>(null);

const selectedUserIdsAtom = atom<Promise<Set<string>>>(async (get) => {
  const selectedUserIds = get(internalSelectedUserIdsAtom);
  if (selectedUserIds !== null) {
    return selectedUserIds;
  }

  // internalSelectedUserIdsAtomがnullの場合、初期状態として扱う。
  // この場合は全ユーザーを選択状態にする
  const users = await get(usersAtom);
  return new Set(users.map((user) => user.id));
});

const toggleUserSelectionAtom = atom(
  null,
  async (get, set, userId: string) => {
    const selectedUserIds = new Set(await get(selectedUserIdsAtom));
    if (selectedUserIds.has(userId)) {
      selectedUserIds.delete(userId);
    } else {
      selectedUserIds.add(userId);
    }
    set(internalSelectedUserIdsAtom, selectedUserIds);
  }
);
```

このコードでは、実際の選択状態を保存するプリミティブatomである`internalSelectedUserIdsAtom`を用意し、中身は`Set<string> | null`型としています。初期状態は`null`です。常にSetが入っているのではなく、初期値を表す特別な値として`null`を使っています。

そして、それをSetに変換する役目を担うのが新しい`selectedUserIdsAtom`です。このatomは、`internalSelectedUserIdsAtom`にすでにSetが入っている場合にはそれを返します。しかし、`internalSelectedUserIdsAtom`が`null`の場合は、初期状態として扱い、`usersAtom`からユーザー一覧を取得して全ユーザーを選択状態にしたSetを返します。

このように、複数のatomを用いて外向きのインターフェース（`Set<string>`）と内部状態（`Set<string> | null`）を分離するテクニックはjotaiでは頻出です。

## 修正後のコードの問題

とはいえ、上記のコードを実際に動かしてみたら、問題があることに気付くはずです。

チェックボックスを1回操作するたびに画面が「Loading...」に戻ってしまいます。

その理由は、`selectedUserIdsAtom`が今や非同期atom（中身がPromiseのatom）となっているからです。内部状態がnullの場合に`usersAtom`の値を使用する必要があり、`usersAtom`は非同期atomなので、`selectedUserIdsAtom`も非同期atomにならざるを得ません。これはasync関数を呼び出してawaitする関数もまたasync関数になるのと同じ理屈です。

確かに上記のコードでも`selectedUserIdsAtom`の読み取り関数がasync関数になっていますね。

チェックボックスを切り替えると`selectedUserIdsAtom`が再評価され、結果が新しいPromiseとなるため、再サスペンドが発生します。今回トランジションは使っていませんので、画面がフォールバックUIに戻ってしまいます。

なお、再サスペンドは本来一瞬で終わるはずです。2回目以降、`usersAtom`（の中身のPromise）はすでに解決済みで、awaitしても一瞬で結果が得られるからです。しかし、現在（v19.2）のReactでは、このケースでは[フォールバックUIが0.3秒表示される仕様になっています](https://github.com/facebook/react/issues/31819)。

### 修正方針

鋭い方は、「なるほど、一瞬フォールバックUIが表示されてしまうことが問題なら、**トランジション**の出番だな」と思われるでしょう。

その考え方は正しいのですが、今回の場合は問題があります。それは、**チェックボックスが制御コンポーネントであるため、チェックボックスの状態変化をトランジションにするのは良くない**ということです。そのため、別の修正方針をとる必要があります。

ここで重要な発想は、「2回目以降は、async関数の都合で非同期処理（Promiseを返す）になっているけど、**本当は非同期処理を行う必要は無い**のではないか」ということです。

最初の1回はユーザーデータの読み込みがあるのでサスペンドが起きるのは必然です。しかし、2回目以降は`usersAtom`の中身は取得済みであり、本来非同期処理を行う必要はありません。したがって、**2回目以降は非同期処理を行わず、同期処理にできればサスペンドを避けられる**という発想になります。

このような「**非同期処理と同期処理の使い分け**」を簡単に実現するためのライブラリが、冒頭で紹介した**jotai-eager**なのです。

## jotai-eagerを使う

さっそく、jotai-eagerを使って先ほどのコードを書き直してみます。jotai-eagerは`eagerAtom`という関数を提供しています。これはjotaiの`atom`関数と同様の使い方で、派生atomを作るものです。

```ts
const selectedUserIdsAtom = eagerAtom((get) => {
  const selectedUserIds = get(internalSelectedUserIdsAtom);
  if (selectedUserIds !== null) {
    return selectedUserIds;
  }

  // internalSelectedUserIdsAtomがnullの場合、初期状態として扱う。
  // この場合は全ユーザーを選択状態にする
  const users = get(usersAtom);
  return new Set(users.map((user) => user.id));
});
```

このように`selectedUserIdsAtom`を書き換えるだけです。これで実行してみると、チェックボックスを切り替えたときに画面がフォールバックUIに戻らなくなったことが分かるはずです。これは、切り替え時に**サスペンドが発生していない**ことを意味します。

では、なぜjotai-eagerを使うとサスペンドが発生しなくなるのでしょうか。仕組みを見てみましょう。

VS Codeなどのエディタを使って`selectedUserIdsAtom`の型を調べると、jotai-eagerの秘密が明らかになります。次のような型になっているはずです。

```ts
Atom<Set<string> | Promise<Set<string>>>
```

つまり、`selectedUserIdsAtom`は`Set<string>`型か`Promise<Set<string>>`型のどちらかを返すatomになっています。

一般化すると、jotai-eagerの`eagerAtom`関数は、**非同期処理が本当に必要な場合のみPromiseを返し、非同期処理が不要な場合は通常の値を返すatomを作成する**のです。

jotai-eagerのポイントは、ReactのSuspenseと同様の仕組みで非同期処理を隠蔽してくれる点にあります。先ほどのコードを、型注釈マシマシで再掲します。

```ts
const selectedUserIdsAtom: Atom<Set<string> | Promise<Set<string>>> = eagerAtom((get): Set<string> => {
  const selectedUserIds: Set<string> | null = get(internalSelectedUserIdsAtom);
  if (selectedUserIds !== null) {
    return selectedUserIds;
  }

  const users: User[] = get(usersAtom);
  return new Set(users.map((user) => user.id));
});
```

すると、2つのポイントが見えてきます。

1つ目のポイントは、`eagerAtom`の読み取り関数は**同期関数**であることです。async関数ではないし、返り値もPromiseではなくただの`Set<string>`です。

では、非同期処理はどこに行ってしまったのでしょうか。実は、`get`に内包されています。これが2つ目のポイントです。

よく見ると、`Promise<User[]>`型を持つ`usersAtom`の値を`get`で取得するところで、結果（`users`の型）が`Promise<User[]>`ではなく`User[]`になっています。ここが`atom`と`eagerAtom`の違いで、普通の`atom`を使っていた場合は`get(usersAtom)`の結果は`Promise<User[]>`型になります。

jotai-eagerの場合は、`get`関数がReactの`use`のような働きをして、Promise入りのatomからPromiseの中身を取り出すことができるのです。

そして、**`get`でPromiseの中身を取り出したときだけ、`eagerAtom`の読み取り結果がPromiseになる**のです。今回の場合、条件分岐の結果次第で結果がPromiseになるかどうか決まります。

```ts
const selectedUserIdsAtom: Atom<Set<string> | Promise<Set<string>>> = eagerAtom((get): Set<string> => {
  // このgetは同期的な読み取り
  const selectedUserIds: Set<string> | null = get(internalSelectedUserIdsAtom);
  if (selectedUserIds !== null) {
    // ここでreturnした場合、非同期処理をしていないので、
    // selectedUserIdsAtomの値はSet<string>になる
    return selectedUserIds;
  }

  // このgetは非同期的な読み取り
  const users: User[] = get(usersAtom);
  // ここでreturnした場合、usersAtomの値を非同期的に取得しているので、
  // selectedUserIdsAtomの値はPromise<Set<string>>になる
  return new Set(users.map((user) => user.id));
});
```

よって、`selectedUserIds`の値がPromiseになるのは初期状態の場合だけで、それ以降（チェックボックスを1回でも操作した後）は`selectedUserIds`の値はPromiseではなく同期的な値になります。

これにより、チェックボックス操作時にサスペンドが発生しなくなります。

### 捕捉: Promiseキャッシュ

実は、今回の場合はjotai-eagerを以下のように使っても目的を達成できます。

```ts
const selectedUserIdsAtom: Atom<Set<string> | Promise<Set<string>>> = eagerAtom((get): Set<string> => {
  const selectedUserIds: Set<string> | null = get(internalSelectedUserIdsAtom);
  const users: User[] = get(usersAtom);

  if (selectedUserIds !== null) {
    return selectedUserIds;
  }

  return new Set(users.map((user) => user.id));
});
```

あまり意味はないけど、`users`の取得を先に持ってきました。こうすると、常に`get(usersAtom)`しているので結局常に`selectedUserIdsAtom`の値がPromiseになってしまうように思えますが、**実はそうなりません**。

なぜなら、jotai-eagerは**一度取得したPromiseの結果をキャッシュする**からです。したがって、2回目以降の`get(usersAtom)`は、初回の計算時に非同期的に取得した`usersAtom`の結果を同期的に再利用できるため、`selectedUserIdsAtom`の値は同期的な値になります。

つまり、今回はjotai-eagerを使うだけで良かったのですね。「解決済みのPromiseの値を取得するだけなのに非同期処理扱いにしなくてもいいのでは？」という感覚をjotai-eagerが実現してくれたのです。

とはいえ、先ほどのように`get(usersAtom)`を条件分岐の後ろに持ってくるほうが、実際の実装としてはおすすめです。それは、派生atomの依存を必要最低限にするためです。

## まとめ

この章では、jotai-eagerを使って非同期処理と必要最低限に抑える方法を説明しました。

jotai-eagerのポイントは、**非同期処理が本当に必要な場合のみPromiseを返し、非同期処理が不要な場合は通常の値を返すatomを作成する**点にあります。`T | Promise<T>`のような型を取り扱う必要があり若干ややこしいですが、うまく使えば無駄なサスペンドを避けられます。

実際のアプリケーションを実装するにあたって、jotai-eagerはとても頼りになるツールです。

### 参考記事

https://zenn.dev/layerx/articles/2676e0c939840e

## 練習問題

この章で学んだjotai-eagerを使って、テーマ切り替え機能を実装してみましょう。

以下のような仕様です。

- ユーザー設定を非同期に取得する`userPreferencesAtom`がある
- テーマの初期値は、ユーザー設定の`preferredTheme`から取得する
- ユーザーは手動でテーマを切り替えられる
- テーマ切り替え時にサスペンドが発生しないようにしたい

以下のコードを完成させてください。

```tsx
import { atom } from "jotai";
import { eagerAtom } from "jotai-eager";

type Theme = "light" | "dark";

interface UserPreferences {
  preferredTheme: Theme;
  // ... 他の設定
}

async function fetchUserPreferences(): Promise<UserPreferences> {
  await new Promise((resolve) => setTimeout(resolve, 500));
  return { preferredTheme: "dark" };
}

const userPreferencesAtom = atom(async () => {
  const preferences = await fetchUserPreferences();
  return preferences;
});

// 現在のテーマを取得するatom
const themeAtom = ???

// テーマを切り替えるatom
const toggleThemeAtom = ???

// 使用例
const ThemeDisplay: React.FC = () => {
  const theme = useAtomValue(themeAtom);

  return (
    <div style={{ background: theme === "dark" ? "#333" : "#fff", color: theme === "dark" ? "#fff" : "#333", padding: "20px" }}>
      現在のテーマ: {theme}
    </div>
  );
};

const ThemeToggleButton: React.FC = () => {
  const toggleTheme = useSetAtom(toggleThemeAtom);

  return (
    <button type="button" onClick={() => toggleTheme()}>
      テーマを切り替える
    </button>
  );
};

const App: React.FC = () => {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <ThemeDisplay />
      <ThemeToggleButton />
    </Suspense>
  );
};
```

:::details 答え

```tsx
// テーマの内部状態（nullは初期状態を表す）
const internalThemeAtom = atom<Theme | null>(null);

// テーマを取得するatom
const themeAtom = eagerAtom((get): Theme => {
  const internalTheme = get(internalThemeAtom);
  if (internalTheme !== null) {
    return internalTheme;
  }

  // 初期状態の場合はユーザー設定から取得
  const preferences = get(userPreferencesAtom);
  return preferences.preferredTheme;
});

// テーマを切り替えるatom
const toggleThemeAtom = atom(null, async (get, set) => {
  const currentTheme = await get(themeAtom);
  const newTheme: Theme = currentTheme === "light" ? "dark" : "light";
  set(internalThemeAtom, newTheme);
});
```

この問題は、本章で学んだパターンと同じ構造になっています。

1. `internalThemeAtom`は`Theme | null`型で、`null`は「まだユーザーが手動で設定していない（初期状態）」を表します
2. `themeAtom`は`eagerAtom`を使って、`internalThemeAtom`がnullの場合のみ`userPreferencesAtom`を参照します
3. `toggleThemeAtom`は`internalThemeAtom`に書き込むことでテーマを切り替えます

`eagerAtom`を使うことで、2回目以降のテーマ切り替えでは`userPreferencesAtom`を参照しないため、`themeAtom`の値はPromiseではなく同期的な値になります。これにより、テーマ切り替え時のサスペンドを防ぐことができます。

:::