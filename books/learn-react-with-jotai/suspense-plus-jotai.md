---
title: "Suspenseとjotaiを組み合わせる"
---

前章では、Suspenseを使用するにあたって、サスペンドするコンポーネントの外にPromiseを持っておく必要があることを説明しました。

前章では親コンポーネントのステートにPromiseを持つ方法を紹介しましたが、この章では別のアプローチとして、**jotai**を使ってPromiseを管理する方法を紹介します。

発想はシンプルです。Promiseをどこかコンポーネントの外に持っておく必要があるのであれば、jotaiのステートとしてPromiseを持てばよい、ということです。

## 派生atomを使って非同期処理を行う

最もシンプルで、実際によく使われる方法は、**派生atom**を使う方法です。派生atomは自身でステートを持たないと説明しましたが、**値のキャッシュ**はされています。そのため、派生atomの読み取り関数の中で非同期処理を行い、Promiseを返すようにすれば、最初の呼び出し時に非同期処理が実行され、その後はキャッシュされた結果が返されるようになります。つまり、Promiseを保管できているわけです。

では、実際にコードを見てみましょう。

```tsx
const userAtom = atom(async (): Promise<User> => {
  const user = await fetchUser();
  return user;
});

const App: React.FC = () => {
  return (
    <Suspense fallback={<p>Loading...</p>}>
      <UserProfile />
    </Suspense>
  );
};

const UserProfile: React.FC = () => {
  const user: User = useAtomValue(userAtom);

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      ...
    </section>
  );
};
```

このコードでは、`userAtom`という派生atomを定義しています。`userAtom`の読み取り関数は非同期関数であり、つまりPromiseを返します。読み取り関数なので`get`を使うことできますが、今回は使っていません。この場合他のステートに依存しない派生atomとなります。

`UserProfile`コンポーネントでは、`useAtomValue`を使って`userAtom`を読み取っています。`userAtom`から読み取れるステートの型は`Promise<User>`ですが、実は`useAtomValue`は`use`を内蔵しており、atomの値がPromiseの場合はサスペンドしてPromiseの中身を返すようになっています。したがって、`user`には`User`型の値が入ります。

ちなみに、この場合`userAtom`の計算は`UserProfile`が初めて`userAtom`を読み取ったときに実行されます。`UserProfile`がレンダリングされるまでは`userAtom`の計算（つまりfetchUserの呼び出し）は行われません。

派生atomの再計算は必要な場合（依存先が変化した場合）にのみ行われます。今回の`userAtom`は他のatomに依存していないため、一度計算されたら再計算されることはありません。つまり、fetchUserは一度だけ呼び出され、その結果がキャッシュされ続けます。

これにより、初回のレンダリング（サスペンドする）とサスペンド明けの再レンダリングで`use`に同じPromiseを与えなければならないという要件が満たされます。

以上が、jotaiを使ってSuspense対応の非同期処理を行う基本的な方法です。

## パラメータ付きの非同期処理

`fetchUser`のような非同期処理は、実際にはパラメータを受け取ることが多いですね。例えばユーザーIDを受け取ってそのユーザーのデータを取得するような場合です。この場合は、上記の方法を少し拡張する必要があります。主に2つの方法をご紹介します。2つはユースケースによって使い分けられます。

### 派生atomにパラメータを渡す方法

1つ目の方法は、同時に1つのパラメータでしか非同期処理を行わない場合に使えます。この場合は、パラメータを保持するためのatomを別途用意し、そのatomに依存する派生atomを作成します。

```tsx
// 取得すべきユーザーIDを保持する
const userIdAtom = atom<string | null>(null);

const userAtom = atom(async (get): Promise<User | null> => {
  const userId = get(userIdAtom);
  // ユーザーIDが設定されていない場合はuserAtomもnullを返す
  if (userId === null) return null;

  const user = await fetchUser(userId);
  return user;
});
```

この例では、`userIdAtom`というプリミティブatomを用意し、そこに取得すべきユーザーIDを保持しています。`userAtom`は`userIdAtom`に依存する派生atomであり、`userIdAtom`の値を取得してから非同期処理を行います。

`userAtom`が取得するユーザーのIDを設定・変更したい場合は、`userIdAtom`に対して値を書き込みます。すると、それに依存した派生atomである`userAtom`が再計算され、再度非同期処理が実行されます。`userAtom`の使用例は以下のようになります。

```tsx
const App: React.FC = () => {
  const setUserId = useSetAtom(userIdAtom);

  const handleSelectUser = (id: string) => {
    setUserId(id);
  };

  return (
    <>
      <UserSelector onSelectUser={handleSelectUser} />
      <Suspense fallback={<p>Loading...</p>}>
        <UserProfile />
      </Suspense>
    </>
  );
};

const UserProfile: React.FC = () => {
  const user: User | null = useAtomValue(userAtom);

  if (user === null) {
    return <p>ユーザーが選択されていません。</p>;
  }

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

これも小さな例ですが、jotaiのアーキテクチャの面白いところが出ています。「取得するユーザーのIDを変えたい」と思った場合、「再度非同期処理を実行する」のような直接的な命令を出すのではなく、「`userIdAtom`の値を書き換える」という形で間接的に指示を出しています。すると、`userIdAtom`に依存する`userAtom`が自動的に再計算され、非同期処理が再度実行されるわけです。

回りくどいと思えるかもしれませんが、これはとてもReactの思想に合ったやり方です。Reactの格言（？）として「UI = f(state)」というものがあるのをご存じの方も多いでしょう。つまり、UIが書き換わるのは、あくまでステート更新の結果として書き換わるのだということです。さらに、Suspenseの世界では、サスペンドという機構を通じて非同期処理ですらその「f」の中に組み込まれています。

あくまで`userIdAtom`が独立したステートなのであって、`userAtom`はそのステートに基づいて計算される派生的なステートに過ぎません。ここが、従来のReactでの非同期処理と結構違うところです。従来は、（実際のステートは`useQuery`のようなライブラリの背後に隠れているかもしれませんが）非同期処理の結果自体がステートとして扱われていました。もちろん、`userId`のようなパラメータがあればそれもステートです。そう考えると、Suspense + jotaiのやり方ではステートの数が減ってシンプルになりましたね。

### atomファクトリーを使う方法

2つ目の方法は、同時に複数のパラメータで非同期処理を行う可能性がある場合に使えます。この場合は、**atomファクトリー**の考え方を使ってパラメータごとにatomを作成します。

```tsx
/**
 * 特定のユーザーIDに対応するuserAtomを作成するatomファクトリー
 */
const createUserAtom = (userId: string) =>
  atom(async (): Promise<User> => {
    const user = await fetchUser(userId);
    return user;
  });

const userAtoms = new Map<string, ReturnType<typeof createUserAtom>>();

const getUserAtom = (userId: string) => {
  let atom = userAtoms.get(userId);
  if (!atom) {
    atom = createUserAtom(userId);
    userAtoms.set(userId, atom);
  }
  return atom;
};

const UserProfile: React.FC<{ userId: string }> = ({ userId }) => {
  const userAtom = getUserAtom(userId);
  const user: User = useAtomValue(userAtom);

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      ...
    </section>
  );
};
```

この例では、`getUserAtom`を使って特定のユーザーIDに対応する`userAtom`を取得しています。`getUserAtom`は、内部で`userAtoms`というMapを使って、既に作成されたatomをキャッシュしています。これにより、同じユーザーIDに対しては同じatomが返されるようになります。

1つ目の方法との違いは、`user1`→`user2`→`user1`のようにユーザーIDが変化した場合に表れます。

- 1つ目の方法: `userAtom`は1つだけなので、再度`user1`になった場合はキャッシュ（`user2`の結果）が使えないため、再計算されfetchが再度実行される
- 2つ目の方法: `user1`用のatomと`user2`用のatomが別々に存在するので、`user1`と`user2`の結果は別々にキャッシュされている。したがって、再度`user1`になった場合は`user1`用のatomのキャッシュが使われ、fetchは再度実行されない

そのため、複数のパラメータが頻繁に切り替わり元に戻る可能性がある場合や、そもそも同時に複数のパラメータで非同期処理を行う可能性がある場合は、2つ目の方法が適しています。

このように、jotaiは必要であればatomを動的に生成して使う文化もあります。atomを`useState`で保持するパターンや、何なら「atoms in atom」としてatomのステートとしてatomを持つパターンもあります。このようなパターンを、変なやり方だと思わずに必要に応じて使いこなせるようになることが重要です。

### atomFamilyを活用する

このようなatomファクトリーのパターンは頻出のため、書きやすくするために`atomFamily`というユーティリティが用意されています。現在（v2時点）ではjotaiに含まれていますが、独立した`jotai-family`パッケージとしても提供されており、こちらの利用が推奨されています。

https://github.com/jotaijs/jotai-family

`atomFamily`を使うと、上記の`createUserAtom`や`getUserAtom`の実装を自前で行う必要がなくなり、以下のように書けます。

```tsx
import { atomFamily } from 'jotai-family';

const userAtomFamily = atomFamily((userId: string) =>
  atom(async (): Promise<User> => {
    const user = await fetchUser(userId);
    return user;
  })
);

const UserProfile: React.FC<{ userId: string }> = ({ userId }) => {
  const userAtom = userAtomFamily(userId);
  const user: User = useAtomValue(userAtom);

  return (
    <section>
      <h1>{user.name}さんのプロフィール</h1>
      ...
    </section>
  );
};
```

内部的には、上述の自分でMapを使ったパターンとやっていることはほとんど変わりません。適宜活用してみてください。

### atomファクトリーパターンの注意点

自分でMapを使う場合も、`atomFamily`を使う場合も共通の注意点があります。それは**メモリ管理**です。jotaiは、atomの値をキャッシュします。そのため、atomを動的に生成してMapなどで保持している場合、そのatomが不要になってもMapから削除しない限りメモリに残り続けてしまい、atom本体に加えてキャッシュされた値も解放されません。

そのため、大量のパラメータでatomを生成するような場合や、サーバー上でjotaiを動かす場合には注意が必要です。`atomFamily`にはタイムスタンプベースで古いatomを自動的に削除できる機能も用意されていますが、これで十分な場合は少なく、場合によっては自前で不要なatomをMapから削除する仕組みを実装する必要があります。