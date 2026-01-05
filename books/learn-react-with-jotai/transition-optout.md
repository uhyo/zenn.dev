---
title: "トランジションのオプトアウト"
---

前2章では、トランジションの基本的な使い方を紹介しました。この章ではその続きとして、トランジションに関連するトピックを説明し、最後にトランジションのオプトアウトについて解説します。

## いつトランジションにすべきか

トランジションは「優先度の低い更新」であると説明しましたね。では、具体的にどのような更新をトランジションにすればよいのでしょうか。

考え方は色々あるとは思いますが、あるReactメンテナは**ほぼ全て**だと述べました。そのため、Reactの意図的には、例外を除いて全部トランジションになっているのがあるべき姿なのでしょう。本当ならデフォルトでステート更新をトランジションにしたかったのかもしれません。現在のように`startTransition`でオプトインする形になっているのは、後方互換性への配慮なのでしょう。

### トランジション内包コンポーネント

このような考え方の帰結として、例えば**汎用のボタンコンポーネントにトランジションを内包させる**ことが考えられます。次のようなイメージです。

```tsx
const MyButton: React.FC<{
  action: () => void;
}> = ({ action, children }) => {
  const [isPending, startTransition] = useTransition();

  const handleClick = () => {
    startTransition(() => {
      action();
    });
  };

  return <button type="button" onClick={handleClick}>
    {isPending ? <LoadingSpinner /> : children}
  </button>;
};

// 使用例
<MyButton action={() => {
  setSomeState(...);
}}>
  状態を更新
</MyButton>
```

ボタンコンポーネントの中に`useTransition`が内包されていますね。このようにしておけば、アプリケーションのどこで使われても、そのボタンが引き起こす状態更新は全てトランジションになります。

さらに、`isPending`を活用することで、ボタン自体にローディングインジケーターを表示することもできます。これも実際のアプリケーションでよく見る挙動です。

ちなみに、ボタンのpropsに着目すると、ボタンが押されたときに呼び出される関数が`action`となっていますね。`onClick`とかにしないの？　と思った方がいるかもしれませんが、実はこの場合は`action`という（あるいは`clickAction`のような）名前にすることがReact公式として推奨されています。これは、ボタンを押されたときに発生するのが**アクション**であることを明示する命名です。

急に**アクション**という新概念が出てきましたが、これはReact 19から打ち出された概念で、話は簡単です。「**トランジション内で実行される関数**」のことをアクションと呼んでいるだけです。

つまり、propsの命名で、「このコールバック関数はトランジション内で実行されますよ」ということをコンポーネントの利用者に伝えているわけです。

## トランジションがネストしたらどうなるのか

基本的に全てをトランジションにする場合、トランジションがネストする可能性があります。その場合はどうなるのでしょうか。

```tsx
const [isPendingA, startTransitionA] = useTransition();
const [isPendingB, startTransitionB] = useTransition();

startTransitionA(() => {
  // トランジションAの中
  startTransitionB(() => {
    // トランジションBの中
    setState(...);
  });
});
```

この場合、`setState`によってサスペンドが発生した場合、`isPendingA`と`isPendingB`が両方とも`true`になります。このように、ステート更新を複数のトランジションに属させることが可能です。

ネストしたトランジションは、先ほどのボタンコンポーネントのようなことをすると発生しがちです。ネストしたトランジションのサポートのおかげで、ボタンコンポーネントは自分専用のトランジションを持って自分の描画のために`isPending`を使うことができるわけですね。

より正確に言えば、現在（v19.2時点）のReactは複数のトランジションが同時並行で進むことをサポートしていません。そのため、このようにトランジションが複数発生した場合、内部的には1つのトランジションに統合される挙動になります。

## トランジションにできないステート更新

先ほど「ほぼ全て」のステート更新をトランジションにすべきだと述べましたが、ではそれに当てはまらないステート更新とはどのようなものでしょうか。

それは、**制御コンポーネントのステートを更新する場合**です。制御コンポーネントとは、例えばフォームの入力欄の値をステートで管理しているようなコンポーネントのことです。

```tsx
const [name, setName] = useState("");

<input
  type="text"
  value={name}
  onChange={(e) => {
    // これはトランジションにできない
    setName(e.target.value);
  }}
/>
```

なぜかというと、制御コンポーネントの`onChange`でのステート更新は「**後追い**」のステート更新である点が特殊だからです。

通常のステート更新は、「プログラム上で`setState`する」→「UIに新しいステートが反映される」という流れをとります。

しかし、制御コンポーネントの場合、ユーザーがinputに入力した時点で**もうDOM上の状態が書き換わっている**のです。ステート更新は、**それに追いつき、整合性を保つ**ために行うものです。

このとき、即座にステート更新をReactの状態に反映する必要があります。そうしないと、「ステートから計算されたUI」と「実際のDOMに表示されているUI」が異なる状態、つまり不整合な状態になってしまいます。これを防ぐために、制御コンポーネントのステート更新はトランジションにしてはいけないのです。

```tsx
const [name, setName] = useState("");

<input
  type="text"
  value={name}
  onChange={(e) => {
    // トランジションにすべきではない
    startTransition(() => {
      setName(e.target.value);
    });
  }}
/>
```

実際にこうしてしまうと、エラーになったりはしませんが、特に`setName`によってサスペンドが発生した場合はユーザーの入力した内容がDOMにすぐ反映されず、無視されたような違和感のある挙動になってしまいます。

## 古いUIが維持される場合とされない場合

これまで、トランジション中にサスペンドが発生した場合の挙動について、「Suspenseのフォールバックが表示される代わりに古いUIが維持される」と説明してきました。

しかし、実はそうではない場合もあります。Reactはトランジションにサスペンドが発生した場合、古いUIを維持するのか、それともフォールバックUIを許容して新しいUIに移行するのかを**自動的に判断**します。

その判断基準は意外とシンプルで、**すでにマウント済みの`Suspense`が再サスペンドしたかどうか**です。再サスペンドの場合は古いUIを維持し、そうでない場合（新規の`Suspense`がサスペンドした場合）はフォールバックになります。複数のサスペンドが発生した場合、再サスペンドが1つでもあれば古いUIが維持されます。

### 新しい`Suspense`がサスペンドした場合

例えば、ユーザーページで最初は「投稿一覧」が表示されていないが、ボタンを押すと投稿一覧が表示されるようになるケースを考えます。

```tsx
function UserPage() {
  const [showPosts, setShowPosts] = useState(false);

  return (
    <div>
      <UserProfile />
      <button onClick={() => {
        startTransition(() => {
          setShowPosts(true);
        });
      }}>投稿を見る</button>
      {showPosts && <Suspense fallback={<LoadingSpinner />}>
        <UserPosts />
      </Suspense>}
    </div>
  );
}
```

この場合、最初は`showPosts`が`false`なので、`UserPosts`はレンダリングされていません。ボタンを押して`showPosts`が`true`になると、新たに`Suspense`がマウントされ、その中で`UserPosts`がサスペンドします。

この場合は**新しい`Suspense`がサスペンドした**ことになるので、ReactはフォールバックUIである`<LoadingSpinner />`を表示します。古いUIを維持する（フォールバックUIが出ず、画面が何も変わらない）わけではありません。

確かに、この状況では古いUIを維持してもあまり意味がありませんね。フォールバックUIが問題となるのは、「サスペンドにより既存の`Suspense`の中身が片付けられて、フォールバックUIに置き換わってしまう」場合です。新規の`Suspense`がマウントされる場合は、もともとその中身は表示されていなかったので、フォールバックUIの弊害が無いわけです。

ただし、書き方がちょっと変わると、同じようなケースでも古いUIが維持される場合があります。今回の場合、`Suspense`自体が条件付きレンダリングされている点がポイントです。次のように書き換えた場合を考えましょう。

```tsx
<Suspense fallback={<LoadingSpinner />}>
  {showPosts && <UserPosts />}
</Suspense>
```

この場合、`showPosts`が`false`のときも`Suspense`はマウントされています。つまり、ボタンを押して`showPosts`が`true`になったときは、**すでにマウント済みの`Suspense`が再サスペンドした**ことになるのです。この場合は、Reactは古いUIを維持します。フォールバックUIは表示されません。

この挙動を念頭に置いて`Suspense`を適切に配置することで、トランジション中のUI挙動をコントロールできます。

厄介に思うかもしれませんが、「Reactは`Suspense`の中身がフォールバックに**置き換わる**のを防いでくれる」と考えればシンプルです。「どこが消えたら困るか」を考えてSuspense境界を配置しましょう。

## トランジションからのオプトアウト

前述の汎用ボタンコンポーネントなどを用いて、ステート更新はトランジションがデフォルトの状態にできたとしましょう。そうなると、新たな問題が出てきます。それは、**敢えてフォールバックUIを出したい場合**です。

古いUIを維持するのではなく、さっさと古いUIを消去して代わりにスケルトンスクリーンでも出して、次のページに遷移しようとしていることをUI上で表現したい場合などですね。先ほど「Reactは古いUIがフォールバックに置き換わるのを防いでくれる」と説明しましたが、場合によっては敢えて置き換えたいわけです。

ここまでの説明から、その方法が分かります。つまり、**既存の`Suspense`を片付けて新規の`Suspense`をサスペンドさせればよい**わけです。

例として、タブ型UIを考えます。

```tsx
const App: React.FC = () => {
  const [activeTab, setActiveTab] = useState<"A" | "B">("A");

  return (
    <div>
      <button onClick={() => {
        startTransition(() => {
          setActiveTab("A");
        });
      }}>タブA {activeTab === "A" ? "(選択中)" : ""}</button>
      <button onClick={() => {
        startTransition(() => {
          setActiveTab("B");
        });
      }}>タブB {activeTab === "B" ? "(選択中)" : ""}</button>

      <Suspense fallback={<LoadingSpinner />}>
        {activeTab === "A" ? (
          <TabAContent />
        ) : (
          <TabBContent />
        )}
      </Suspense>
    </div>
  );
};
```

この場合、タブ切り替えの際に`Suspense`内で再サスペンドが発生するため、古いUIが維持されます。

このUIの場合に問題なのは、古いUIが維持されるために、タブBを押してもすぐに「タブB（選択中）」にならない点です。今回実現したいUIが「タブBを押したらすぐにタブB（選択中）になる」ことである場合、古いUIを維持する挙動は望ましくありません。

そこで、タブを切り替えたときは前のタブの古い内容を維持するのではなく新しいタブのフォールバックUIを表示したいとします。そのためには、次のように書き換えます。

```tsx
const App: React.FC = () => {
  const [activeTab, setActiveTab] = useState<"A" | "B">("A");

  return (
    <div>
      <button onClick={() => {
        startTransition(() => {
          setActiveTab("A");
        });
      }}>タブA {activeTab === "A" ? "(選択中)" : ""}</button>
      <button onClick={() => {
        startTransition(() => {
          setActiveTab("B");
        });
      }}>タブB {activeTab === "B" ? "(選択中)" : ""}</button>

      {activeTab === "A" && (
        <Suspense fallback={<LoadingSpinner />}>
          <TabAContent />
        </Suspense>
      )}
      {activeTab === "B" && (
        <Suspense fallback={<LoadingSpinner />}>
          <TabBContent />
        </Suspense>
      )}
    </div>
  );
};
```

先ほどとの違いは、`Suspense`を2つに分けて、それぞれを条件付きレンダリングしている点です。こうすると、タブAの`Suspense`とタブBの`Suspense`は**別々のインスタンス**になります。

そのため、タブを切り替えたときは**既存の`Suspense`が片付けられ、新しい`Suspense`が出現と同時にサスペンドする**ことになります。

これは古いUIが維持される条件に当てはまらないので、ReactはフォールバックUIを表示します。これにより、タブ切り替え時にすぐに「タブB（選択中）」が表示され、かつ`LoadingSpinner`も表示されるようになります。

また、同様の効果を得るために、`key`属性を用いる方法もあります。例えば次のようにします。

```tsx
<Suspense key={activeTab} fallback={<LoadingSpinner />}>
  {activeTab === "A" ? (
    <TabAContent />
  ) : (
    <TabBContent />
  )}
</Suspense>
```

この場合も、`activeTab`が変わるたびに`Suspense`のインスタンスが変わるため、同様にフォールバックUIが表示されます。

このような`key`の使い方については以下の記事で詳しく解説していますので、参考にしてください。

https://zenn.dev/uhyo/articles/react-key-techniques

## まとめ

この章では、トランジションの詳細な挙動や、Reactアプリにおけるトランジションの考え方を説明しました。

また、その知識を利用して、トランジションをオプトアウトする方法も解説しました。

## 練習問題

この章で学んだ「トランジションのオプトアウト」を実践してみましょう。

以下のアプリケーションは、シンプルなページナビゲーションです。「ホーム」「プロフィール」「設定」の3つのページがあり、それぞれボタンを押すと切り替わります。

```tsx
type PageId = "home" | "profile" | "settings";

async function fetchPageContent(pageId: PageId): Promise<string> {
  await new Promise((resolve) => setTimeout(resolve, 500));
  const contents: Record<PageId, string> = {
    home: "ホームページへようこそ！",
    profile: "あなたのプロフィール情報です。",
    settings: "設定を変更できます。",
  };
  return contents[pageId];
}

const pageContentAtomFamily = atomFamily((pageId: PageId) =>
  atom(async () => fetchPageContent(pageId))
);

const App: React.FC = () => {
  const [currentPage, setCurrentPage] = useState<PageId>("home");

  const handleNavigate = (pageId: PageId) => {
    startTransition(() => {
      setCurrentPage(pageId);
    });
  };

  return (
    <div>
      <nav>
        <button type="button" onClick={() => handleNavigate("home")}>
          ホーム {currentPage === "home" ? "(現在)" : ""}
        </button>
        <button type="button" onClick={() => handleNavigate("profile")}>
          プロフィール {currentPage === "profile" ? "(現在)" : ""}
        </button>
        <button type="button" onClick={() => handleNavigate("settings")}>
          設定 {currentPage === "settings" ? "(現在)" : ""}
        </button>
      </nav>

      <Suspense fallback={<p>ページを読み込み中...</p>}>
        <PageContent pageId={currentPage} />
      </Suspense>
    </div>
  );
};

const PageContent: React.FC<{ pageId: PageId }> = ({ pageId }) => {
  const content = useAtomValue(pageContentAtomFamily(pageId));
  return <main><p>{content}</p></main>;
};
```

現在の実装では、ページを切り替えるとトランジションにより古いページの内容が表示され続け、新しいページの読み込みが完了してから切り替わります。

しかし、このページナビゲーションでは**すぐにフォールバックUIを表示して、ページが切り替わったことをユーザーに伝えたい**とします。この章で学んだテクニックを使って、ページ切り替え時にフォールバックUI（「ページを読み込み中...」）が表示されるように修正してください。

:::details 答え

```tsx
const App: React.FC = () => {
  const [currentPage, setCurrentPage] = useState<PageId>("home");

  const handleNavigate = (pageId: PageId) => {
    startTransition(() => {
      setCurrentPage(pageId);
    });
  };

  return (
    <div>
      <nav>
        <button type="button" onClick={() => handleNavigate("home")}>
          ホーム {currentPage === "home" ? "(現在)" : ""}
        </button>
        <button type="button" onClick={() => handleNavigate("profile")}>
          プロフィール {currentPage === "profile" ? "(現在)" : ""}
        </button>
        <button type="button" onClick={() => handleNavigate("settings")}>
          設定 {currentPage === "settings" ? "(現在)" : ""}
        </button>
      </nav>

      <Suspense key={currentPage} fallback={<p>ページを読み込み中...</p>}>
        <PageContent pageId={currentPage} />
      </Suspense>
    </div>
  );
};
```

ポイントは、`Suspense`に`key={currentPage}`を追加したことです。

`key`属性が変わると、Reactはその要素を**別のインスタンス**として扱います。つまり、`currentPage`が`"home"`から`"profile"`に変わったとき、「ホーム用のSuspense」が片付けられ、「プロフィール用のSuspense」が新しくマウントされることになります。

これは「すでにマウント済みのSuspenseが再サスペンドする」のではなく「新しいSuspenseがサスペンドする」ケースに該当するため、Reactは古いUIを維持せずにフォールバックUIを表示します。
:::