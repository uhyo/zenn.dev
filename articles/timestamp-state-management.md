---
title: "王道か邪道か？　タイムスタンプによるステート管理"
emoji: "⏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

Reactによるステート管理では、ある状態が変化したら付随して他の状態も変化してほしい場合があります。例えば、次のような場合を考えます。

- チェックボックスが1つある。
- チェックボックスの初期状態は、HTTP APIから取得したデータによって決まる。
- ユーザーはチェックボックスを操作できる。
- APIからデータを再取得する場合があり、その場合はチェックボックスの状態が再取得されたデータに従ってリセットされる。

皆さんは、このような要件をどのように実装するでしょうか。

## やりがちな実装

まず、やりがちな実装を見てみましょう。

```tsx
const apiData = useApiData();
const [isChecked, setIsChecked] = useState(false);

useEffect(() => {
  setIsChecked(apiData.isChecked);
}, [apiData.isChecked]);
```

これは、GitHub Copilotが上記の要件に基づいて書いたコードです。同じようなコードを書いたことがある人も多いのではないでしょうか。

言うまでもなく、このようなコードを書いた人はuseEffect警察に連れ去られます。

値の変化に反応するためにuseEffectを使うのは良くありません。この例の場合、仕様通りの状態にたどり着くまでに2回のレンダリングが必要になり、パフォーマンスの問題が発生したり、バグの温床になったりします。後者は本当に深刻で、useEffectの3連鎖みたいなコードの中にバグが混ざっているとデバッグが本当にしんどいです。

## 良さそうな実装

それでは、どのように実装すればよいのでしょうか。典型的な解決策は、実は`useApiData`が`onSuccess`みたいな機能を持っていて、データ取得に成功したときの処理が書けるようになっていることです。

```tsx
const apiData = useApiData({
  onSuccess: (data) => {
    setIsChecked(data.isChecked);
  },
});
const [isChecked, setIsChecked] = useState(apiData.isChecked);
```

一般に、良くないuseEffectの使い方の解消法として、useEffectの処理をイベントハンドラに移動するという方法があります。これも似たような解消法だと考えられます。

しかし、このような方法が取れない場合もあるでしょう。例えば、[TanStack Query](https://tanstack.com/query/latest)を使っていて`onSuccess`のようなAPIが無い場合とか、実際には`apiData`と`isChecked`の定義が別々の場所にあり、`useApiData`から`setIsChecked`を参照するのは難しい場合とかです。

## タイムスタンプによるステート管理

このような場合に1つの方法として考慮に値するのが、タイムスタンプによるステート管理です。

つまり、`apiData`と`isChecked`がそれぞれタイムスタンプを持つことで、「どちらが後に更新されたのか」という情報を得ることができます。これによって、`apiData`（APIから取得したデータ）と`isChecked`（ユーザー操作による状態）のどちらが優先されるべきかを判断することができます。

```tsx
type Timestamped<T> = { data: T; timestamp: number };

const apiData: Timestamped<ApiData> = useApiData();
const [userChecked, setUserChecked] = useState<Timestamped<boolean>>({
  data: false
  timestamp: 0,
});

const isChecked = userChecked.timestamp > apiData.timestamp
  ? userChecked.data
  : apiData.data.isChecked;
```

この場合、次のような動作となります。

- 初期状態（APIからデータを取得後）は、`isChecked`にはAPIから取得したデータが反映される。
- ユーザーがチェックボックスを操作すると、`userChecked`が更新されてそちらのタイムスタンプが新しくなるので、`isChecked`はユーザー操作による状態が反映される。
- APIからデータを再取得すると、`apiData`が更新されてそちらのタイムスタンプが新しくなるので、ユーザー操作の結果は無視され、`isChecked`はAPIから取得したデータが反映される。

この方法は、`useEffect`を使うことなく、それぞれの状態を独立したものとして扱うことができます。一般に、ステート管理においてはその対象を最小限とし、計算可能なものは計算によって求めることが望ましいとされています。タイムスタンプを導入することで、`isChecked`を単なる計算結果として扱うことができるようになりました。

## Jotaiと相性がいい

このようなやり方は、[Jotai](https://jotai.org/)のようなステート管理アーキテクチャと相性がいいと考えています。

Jotaiでは個々のステートをatomとして表現するほか、derived atom（他のatomから計算されるatom）を使うことができます。これを応用することで、既存のatomをラップしてタイムスタンプ付きのatomを作ることができます。キャッシュなどもう少し調整が必要なものの、雑にアイデアを書くと次のようになります。

```tsx
function timestampedAtom<T>(atom: Atom<T>): Atom<Timestamped<T>> {
  return atom((get) => {
    return {
      data: get(atom),
      timestamp: performance.now(),
    };
  });
}
```

この方法だと、タイムスタンプを必要とする側で`timestampedAtom`を使うだけで、ステート管理に必要なタイムスタンプを得ることができるでしょう。

## まとめ

Reactなどでは、保持するステートを最小限にして、他の状態は計算によって求めるのがベストプラクティスとされています。この記事では、このやり方を徹底するためにタイムスタンプを利用できることを紹介しました。

筆者は実際このやり方を（Recoilと一緒に）使ったことがありますが、さすがに邪道なのでは？　という思いもあります。皆さんの感想をお聞かせください。