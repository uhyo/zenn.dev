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

しかし、この方法には不安な点があります。「`apiData`が変わるステート更新」と、「`onSuccess`内の`setIsChecked`」が同時に行われないと、1回のレンダリングで変化してくれず、useEffectと同じ問題が残ってしまいます。そうなっているかどうかは`useApiData`の内部実装次第であり、他のモジュールの内部実装にむやみに依存してしまうという気持ち悪さがあります。

理想的には、`apiData`が変わったということが新しいレンダリング結果にどう反映されるのかより明快なロジックにしたいです。

また、このような方法が取れない場合もあるでしょう。例えば、[TanStack Query](https://tanstack.com/query/latest)を使っていて`onSuccess`のようなAPIが無い場合とか、実際には`apiData`と`isChecked`の定義が別々の場所にあり、`useApiData`から`setIsChecked`を参照するのは難しい場合とかです。

## タイムスタンプによるステート管理

このような場合に1つの方法として考慮に値するのが、タイムスタンプによるステート管理です。

つまり、APIから取得したデータを表す`apiData`とユーザーのチェック操作の結果を表す`userChecked`を用意し、さらにそれらがタイムスタンプを持つようにします。こうすることで、「どちらが後に更新されたのか」という情報を得ることができます。

最終的なチェックボックスの状態を表す`isChecked`は、タイムスタンプを見て`apiData`と`userChecked`のどちらが優先されるべきかを判断することで求めることができます。

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

この方法では、`userChecked`というステートを導入したことで、APIのデータとユーザーの操作を独立した（互いに干渉しない）状態として扱うことができました。さらにタイムスタンプを導入することで、`isChecked`は単なる計算結果として扱うことができるようになりました。

一般に、ステート管理においてはその対象を最小限とし、計算可能なものは計算によって求めることが望ましいとされています。この設計であれば、ステート更新の意味が「APIのデータが更新された」「ユーザーがチェックボックスを操作した」という明確なものになり、最終的なUIの状態はそれらのステートから計算されるという理想的な設計になっていると言えるでしょう。

:::message
時刻を使ったら巻き戻ったりしたときに困るのでは？　という疑問をいただきました。実は、`performance.now()`を使ってタイムスタンプを取得すれば巻き戻らないことが保証されています。`Date.now()`だと巻き戻りが発生する可能性があるので、この用途だと`performance.now()`が適しているでしょう。

その代わり、この記事の内容はクライアント側（ブラウザ内）に閉じた話となります。サーバーが発行したタイムスタンプとクライアントの時刻を組み合わせるようなことはできません。
:::

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