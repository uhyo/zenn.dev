---
title: "Reactのデータ再取得、タイムスタンプで管理すると宣言的になる話【AI生成】"
emoji: "⏰"
type: "tech"
topics: ["react", "typescript"]
published: true
---

:::details AI生成記事について

この記事は、AIが生成した記事を無修正で公開しています。投稿者（人間）の普段の作風・意見と異なる点や内容の粗もありますが、技術記事として公開するに足るクオリティであるという投稿者の判断と責任により投稿するものです。

ただし、記事に含まれる、経験に基づくエピソードは全てAIによる創造である点はご了承ください。

ちなみに、記事の生成は[事前に用意したスタイルガイド](https://zenn.dev/uhyo/articles/ai-writes-tech-articles-202510)に基づき、人間が記事のアイデアと結論を与えてAI (Claude Code) が出力したものであり、出力結果に対する追加の修正依頼などは行わない一発撮りです。

:::

皆さんこんにちは。最近、データ再取得の実装パターンについて考えていて、面白い気づきがあったので共有します。

Reactでデータを再取得したいとき、普通は`refetch()`みたいな関数を呼ぶ実装になります。ボタンを押したらサーバーからデータを取り直す、みたいなやつです。でも、これって結構命令的というか、Suspense時代のReactの思想とずれてる気がするんです。

## よくある実装の違和感

典型的な実装、こんな感じで書いたことある人多いと思います。

```typescript
function UserProfile() {
  const { data, refetch } = useQuery('/api/user');

  return (
    <div>
      <UserInfo data={data} />
      <button onClick={() => refetch()}>更新</button>
    </div>
  );
}
```

これ、動くし問題ないです。でも、なんか気持ち悪さがあって。「ボタンを押す → 関数を呼ぶ → データが更新される」という流れが、すごく手続き的なんです。Reactの宣言的なUIという思想からすると、ちょっと違和感があります。

特にSuspenseと組み合わせると、この違和感が際立ちます。Suspenseって、データ取得を宣言的に扱うための仕組みじゃないですか。「このコンポーネントはこのデータが必要」って宣言すれば、あとはReactがよしなにやってくれます。なのに、再取得だけ命令的に`refetch()`を呼ぶのって、なんか一貫性がない気がします。

筆者、前に似たようなこと考えてたプロジェクトがあって（確か社内のだったかな？）、その時はあんまり深く考えずrefetch使っていました。でも今思うと、もっといい方法があったかもしれません。

## タイムスタンプで管理する発想

最近気づいたのが、**タイムスタンプをstateとして持つ**というアプローチです。具体的にはこんな感じ。

```typescript
function UserProfile() {
  const [refreshKey, setRefreshKey] = useState(Date.now());
  const { data } = useQuery('/api/user', { refreshKey });

  return (
    <div>
      <UserInfo data={data} />
      <button onClick={() => setRefreshKey(Date.now())}>更新</button>
    </div>
  );
}
```

何が変わったかというと、再取得の「トリガー」を状態として表現しています。ボタンを押すと`refreshKey`が更新されて、それが`useQuery`の依存配列に入ってるから、自動的にデータ取得が走ります。

これ、一見するとただの回りくどいrefetchに見えるかもしれません。でも、考え方として結構違います。**「今のUIのバージョン」**という概念を導入してるんです。

## UIのバージョンという考え方

この`refreshKey`、実質的には「今表示してるUIが、どのタイミングのデータを基にしてるか」を表しています。タイムスタンプが古いUIと新しいUIは、別々のバージョンとして扱えます。

```typescript
// 例えば、refreshKeyを使って「最終更新時刻」を表示できる
function UserProfile() {
  const [refreshKey, setRefreshKey] = useState(Date.now());
  const { data } = useQuery('/api/user', { refreshKey });

  return (
    <div>
      <UserInfo data={data} />
      <p>最終更新: {new Date(refreshKey).toLocaleString()}</p>
      <button onClick={() => setRefreshKey(Date.now())}>更新</button>
    </div>
  );
}
```

これ、ただのrefetchだと実装が面倒になります。更新時刻を別のstateで持たないといけないし、refetchと同時に更新時刻も更新する必要があります。でもタイムスタンプを使うと、そもそも更新タイミングがstateになってるから、そのまま表示できます。

筆者はこれを「構造的バージョニング」と呼んでいます（勝手に名付けました）。UIの状態を時系列のバージョンとして捉えることで、データ取得が「新しいバージョンのUIに必要なデータを取る」という宣言的な処理になります。

## Suspenseとの相性

このパターン、React 18のSuspenseと組み合わせると特に輝きます。

```typescript
function UserProfile() {
  const [refreshKey, setRefreshKey] = useState(Date.now());

  return (
    <div>
      <Suspense fallback={<Loading />}>
        <UserData refreshKey={refreshKey} />
      </Suspense>
      <button onClick={() => setRefreshKey(Date.now())}>更新</button>
    </div>
  );
}

function UserData({ refreshKey }: { refreshKey: number }) {
  // SuspenseをサポートするデータフックAPI (SWRとかReact Queryとか)
  const data = useSuspenseQuery('/api/user', { refreshKey });

  return <UserInfo data={data} />;
}
```

ボタンを押すと、`refreshKey`が更新されます。それに伴って`UserData`が再レンダリングされて、新しい`refreshKey`でデータ取得が走ります。この間、Suspenseが自動的にローディング状態を表示してくれます。

これ、すごく宣言的じゃないですか。「refreshKeyが変わったから、そのバージョンに対応するデータを取得して、取得中はローディングを表示する」という一連の流れが、Reactの仕組みに乗っかって自然に実現できます。

`refetch()`を使う場合、ローディング状態の管理が面倒になります。多くのライブラリは`isRefetching`みたいなフラグを提供してるけど、それを手動でローディングUIに繋げないといけません。あ、これバグりそうだな...ローディング中に複数回ボタン押されたらどうなるんだろ。まあ、ライブラリが勝手にハンドリングしてくれるか。

## 実装例：複数の更新トリガー

このパターンの強みは、複数の更新トリガーを統一的に扱えることです。

```typescript
function Dashboard() {
  const [dataVersion, setDataVersion] = useState(Date.now());

  // 定期的な自動更新
  useEffect(() => {
    const timer = setInterval(() => {
      setDataVersion(Date.now());
    }, 60000); // 1分ごと

    return () => clearInterval(timer);
  }, []);

  // 手動更新
  const handleManualRefresh = () => {
    setDataVersion(Date.now());
  };

  // WebSocketでの更新通知
  useWebSocket('ws://api/updates', {
    onMessage: (event) => {
      if (event.data.type === 'data-changed') {
        setDataVersion(Date.now());
      }
    }
  });

  return (
    <div>
      <DataDisplay dataVersion={dataVersion} />
      <button onClick={handleManualRefresh}>今すぐ更新</button>
    </div>
  );
}
```

タイマー、手動ボタン、WebSocket、どれも同じ`dataVersion`を更新するだけです。データ取得ロジックは`DataDisplay`内に閉じてて、外側は「いつデータを取得するか」だけを制御しています。責任の分離ができて綺麗です。

refetchだとこうはいきません。各トリガーごとに`refetch()`を呼ぶ必要があって、データ取得の実装とトリガーの実装が密結合になります。まあ、関数を変数に入れておけば似たようなことはできるけど、なんかこう、しっくりこないんです。

## 注意点とか

このパターン、万能じゃありません。いくつか気をつけないといけない点があります。

まず、タイムスタンプが衝突する可能性があります。連打されると同じミリ秒で複数回更新される可能性があります。まあ、実際には問題にならないことが多いけど、厳密にやりたい場合はインクリメンタルなカウンターを使うほうがいいかもしれません。

```typescript
const [refreshKey, setRefreshKey] = useState(0);
// ...
onClick={() => setRefreshKey(prev => prev + 1)}
```

これならタイムスタンプの意味はなくなるけど、「バージョン」という概念は残ります。筆者はタイムスタンプ派ですが（最終更新時刻が表示できるから）、カウンターのほうが堅実かもしれません。

あと、データフックライブラリによっては、このパターンをサポートしてないこともあります。ライブラリが依存配列をちゃんと見てくれないと、タイムスタンプを更新してもデータ取得が走りません。SWRとかReact Queryは対応してるけど、自前のhookだと実装が必要です。

そういえば、この前Twitterで似たような話を見た気がします。「状態を時系列で管理する」みたいな話だったかな？うろ覚えだけど。

## まとめ

Reactのデータ再取得、`refetch()`みたいな命令的なAPIじゃなくて、タイムスタンプやバージョン番号をstateで管理すると宣言的になります。特にSuspenseと組み合わせると、「新しいバージョンのUIを表示する → そのバージョンに必要なデータを取得する」という流れが自然に表現できます。

まだ全部のプロジェクトでこのパターンを試したわけじゃないけど（最近気づいたばっかりなので）、個人的には結構気に入っています。複数の更新トリガーを統一的に扱えるのも便利だし、コードの見通しが良くなる感じがします。

そのうち、もっと複雑なケース（楽観的更新とか）でも試してみたいですね。
