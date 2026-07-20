---
title: "Async React時代の宣言的UI 3: useActionStateでユーザーの操作を妨げないUXを作る"
emoji: "🧵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

**useActionState**は、React 19で導入された新しいフックです。

その挙動を見ると、非同期版useReducerと言えるような挙動をします。

ただ、筆者は最近、このフックの真髄は**ユーザーの操作を妨げないUX**を作ることにあるのではないかと考えています。ReactはUIライブラリであり、そのゴールは「良いUX」であるように思います。

そこで、この記事では、useActionStateがどのようなUXを実現するのに役立つのか、という観点で解説していきます。

## useActionStateのインターフェース

まずuseActionStateのインターフェースを見ましょう。[ドキュメント](https://ja.react.dev/reference/react/useActionState)によると、インターフェースはこうなっています。

```ts
const [state, dispatchAction, isPending] =  useActionState(reducerAction, initialState, permalink?);
```

確かに`reducer`とか`dispatch`といった語彙が見えるので、useReducerに近いもののようです。

違うところは、`reducerAction`は**非同期関数でもよく、副作用があってもよい**という点です。

実際、公式ドキュメントのuseActionStateの説明は以下のようになっています。

> useActionState は、[アクション (Action)](https://ja.react.dev/reference/react/useTransition#functions-called-in-starttransition-are-called-actions) を使って副作用を伴う state 更新を行うための React フックです。

「副作用を伴うstate更新」とありますね。典型的には、サーバーにPOSTリクエストを送るといったことが副作用にあたります。

使用例を公式ドキュメントから引用します。

```ts
const [count, dispatchAction, isPending] = useActionState(async (prevCount) => {
  return await addToCart(prevCount)
}, 0);

// API呼び出しのつもり
export async function addToCart(count) {
  await new Promise(resolve => setTimeout(resolve, 1000));
  return count + 1;
}
```

この例では、`count`は初期値0です。ボタンを押すなどして`dispatchAction`を呼ぶと、アクションが発火して、`addToCart`が呼ばれます。これはサーバーと通信してサーバー側のカートに追加する処理を行うのでしょう。その結果として、サーバーから最新状態（この場合はカートの中身の数）が返ってきて、`count`が更新されます。

## アクションのキューイング

顕著な`useActionState`の特徴として、**アクションがキューイングされる**ことが挙げられます。

つまり、前の`dispatchAction`がまだ終了していない状態で次の`dispatchAction`が呼ばれた場合、2つ非同期処理（`reducerAction`）が並行して走るのではなく、前の処理が終わるまで待ってから次の処理が走る、という挙動になります。

これはドキュメントにも明記された、意図した挙動です。ちなみに`useTransition`にはこの挙動はなく、`useActionState`に特有のものです。

これは、半分はインターフェース設計上の必然であり、もう半分は良いUXを簡単に作れるようにするためだと筆者は考えています。

前者は、`reducerAction`が前の状態を引数として受け取るからです。副作用1→副作用2という順番で処理が走る場合、副作用2は副作用1の結果を受け取る必要があります[^note_parallel]。そのためには、必然的に、副作用1の結果が出ないと副作用2は実行できない、ということになります。

[^note_parallel]: もし並行実行を許すと、このインターフェースでは破綻してしまいますね。並行して走った副作用1と副作用2の結果をどうマージするか？　といったことを考慮する必要があります。React的には、そのような場合は`useActionState`を使わずに自分でやってねということでしょう。

後者がこの記事の主題です。以降で詳しく見ていきましょう。

## ショッピングカートの例に見る良いUX

現実のUIでは、特にサーバーへのPOSTなど副作用を伴う場合、副作用の順番が重要な場面が多いでしょう。例えば、カートに商品を追加する場合、ユーザーが「商品Aを追加」→「商品Bを追加」と操作した場合、カートの並び順は「商品A→商品B」でなければなりません。もし副作用が並行して走ると、サーバー側で「商品B→商品A」となってしまう可能性があります[^note_server_api]。

[^note_server_api]: クライアントからユーザー操作のタイムスタンプを送るとか色々工夫しようはあるのですが、そういう変な工夫は想定しないことにしましょう。

また、カートに追加APIを呼び出している最中にユーザーが「注文確定」ボタンを押した場合、最後に追加した商品が注文に含まれるのかどうか怪しくなりますね。

このように、サーバーのAPIをむやみやたらに並列に呼び出してはいけない場合、どのように実装するでしょうか。

一番典型的な方法は、ユーザーの操作をブロックすることです。例えば、カートに商品を追加するボタンを押したら、サーバーからレスポンスが返ってくるまでボタンを無効化してしまう、といった方法です。皆さんもそういう実装をした経験はありませんか？

しかし、これはUXに改善の余地があります。特にユーザーがカートに複数の商品を追加したい場合、カートに商品を1つ追加するごとにボタンがブロックされるのは望ましくありません。ユーザーはAPI待ちなんか知ったことではなく、次々と商品を追加したいはずです。ユーザーの操作をブロックするのは、UXとしてはあまり良くありません。

それを解決する方法が、まさにキューイング（＋楽観的更新）です。ユーザーには快適に操作させておいて、呼び出すべきAPIを裏でキューイングしておけばいいのです。

ユーザーが経験する待ち時間が変わるわけではありませんが、操作の1つ1つをブロックされ細かく待たされるのに比べればユーザーのストレスが軽減されるでしょう。これが良いUXです。

そして、useActionStateはこのキューイングを簡単に実現してくれます。まさに今説明したような挙動が、Reactに組み込まれているのです。

この「Async React時代の宣言的UI」シリーズでは、Async Reactと呼ばれる一連の機能が、普通に実装すれば良いUXになるように設計されているということを解説してきました。

今回のuseActionStateもその一例です。キューイングとかを自ら実装しなくても、useActionStateを使えば自動的にそうなります。そして、先ほど説明したように、キューイングの挙動はuseActionStateのインターフェースからの必然でもあります。

これは、非同期的な副作用はキューイングしないとだめだよね、というReactのオピニオンが反映された結果とも言えます。Async React時代のReactは良いUXのために結構オピニオンを出してくるイメージがあります。

元々useActionStateのインターフェースは過度にキューイングを意識させるものではなく、「副作用を含むことができる非同期なreducer」として自然な形になっています。そこにうまくキューイングの処理を巻き込むことで、使いやすいフックであることと良いUXを両立しています。

筆者としては、useActionStateもまた、Async React時代の宣言的UIの一例であると考えています。もちろんuseActionStateをキューイング目当てで使ってよいのですが、コードの表面上はあくまで副作用を含む非同期処理を交えたステート管理、という形になっています。キューイングという挙動を直接指示するのではなく、インターフェースからの演繹としてキューイングが実現されるのです。

## useActionStateを使う実装の具体例

ということで、具体的な実装例を見てみましょう。ショッピングカートの例です。実際に動作しているところは以下のStackBlitzで確認できます。

https://stackblitz.com/~/github.com/uhyo/useactionstate-queue-trick

以下のような画面が表示されます。

![画面左半分に商品一覧と「Add to cart」ボタンがある。右半分にはカートの中身が表示されており、「Order Now」ボタンがある。](/images/async-react-action-queue/sample-1.png)

「Add to cart」ボタンを押すとカートに商品が追加されます。画面上すぐ追加されますが、実際にはAPIリクエストが発行されており、処理完了まで時間がかかります。その間はカートの表示が半透明になり、楽観的更新であることを示しています。その状態でも、「Add to cart」ボタンは次々と押すことができ、カートには即座に商品が追加されております。全てのAPIリクエストが完了すると、カートの内容が確定し、表示が通常の不透明な状態に戻ります。

「Order Now」ボタンも、カートの内容が空でない限り押すことができます（楽観的更新によりカートの中身がある表示になった時点ですでに押せます）。カートの追加処理が途中の状態で「Order Now」ボタンを押した場合は、カートの追加処理と注文処理が同じキューに入るので、ユーザーが追加したものは全てカートに追加された状態で注文処理が走ります。

これなら、ユーザーが注文ボタンを押すまでの間は、ユーザーの操作を妨げるものはありません。良いUXですね。

実装を見ましょう。キーとなる`useActionState`の部分は以下のようになっています（実際のコードから少し簡略化しています）。

```ts
const [cartState, dispatch, isPending] = useActionState(
  async (prev: CartState, action: CartAction): Promise<CartState> => {
    switch (action.type) {
      case 'add': {
        const next = await addToCartApi(action.product)
        return { cart: next, lastOrder: prev.lastOrder }
      }
      case 'order': {
        const result = await orderApi()
        return { cart: [], lastOrder: result }
      }
    }
  },
  { cart: [], lastOrder: null },
);
```

1つのuseActionStateで、カートに商品を追加する処理と注文処理の両方を扱っています。非同期reducerの結果は、常にAPIの結果を反映した最新の状態になります（`prev.lastOrder`だけAPIと関係ないので引き継いでいますが）。シンプルで直観的ですね。

そしてやはり、useActionStateはuseOptimisticと組み合わせて使うのがよいですね。今回はこうです。

```ts
const [optimistic, dispatchOptimistic] = useOptimistic<
  OptimisticCartState,
  CartAction
>(
  cartState,
  (state: OptimisticCartState, action: CartAction): OptimisticCartState => {
    switch (action.type) {
      case 'add': {
        const existing = state.cart.find(
          (i) => i.product.id === action.product.id,
        )
        const cart = existing
          ? state.cart.map((i) =>
              i.product.id === action.product.id
                ? { ...i, quantity: i.quantity + 1, pending: true }
                : i,
            )
          : [
              ...state.cart,
              { product: action.product, quantity: 1, pending: true },
            ]
        return { ...state, cart }
      }
      case 'order':
        return { ...state, ordering: true }
    }
  },
);
```

楽観的更新は、サーバー側で行われるであろう処理をフロントエンドでも再現することで、結果をすぐフィードバックします。このコードでは実際のreducerに渡すのと同じ`CartAction`を受け取り、結果を計算するという理想的な形になっています。

ただし、楽観的更新であることが分かるフラグ付けをしています。上のコードだと、`pending: true`や`ordering: true`がそれに当たります。これにより、UI上で半透明表示などの演出を行うことができます。このような`useOptimistic`の使い方（サーバーの状態を再現するだけでなくUIの演出のために行う）は頻出パターンです。トランジションが完了すると自動的に楽観的更新の効果が消えることから、処理中といった状態を表現するのに便利です。

今回の場合、`optimistic.ordering`（注文処理中）の場合は「Add to cart」ボタンを押せないようにするといった制御が入っています。

それらを呼び出すイベントハンドラは以下のようになっています。シンプルですね。

```ts
const handleAdd = (product: Product) => {
  startTransition(() => {
    dispatchOptimistic({ type: 'add', product })
    dispatch({ type: 'add', product })
  })
}

const handleOrder = () => {
  startTransition(() => {
    dispatchOptimistic({ type: 'order' })
    dispatch({ type: 'order' })
  })
}
```

## まとめ

終わってみれば、`useActionState`の基本的な使い方 + `useOptimistic`のちょっとした応用を紹介する記事になりました。

読者の皆さんに持ち帰っていただきたいのは、**処理中だからといってユーザーの操作をブロックするのではなく、楽観的更新と組み合わせてユーザーの操作を妨げないという意識**です。

自前で完璧に実装しようとすると面倒くさいかもしれませんが、そのための道具はReactが提供してくれています。

プロダクト開発では、知識があるかどうかによって仕様の判断が変わることがあります。自前でキューイングシステムを備えるのは嫌だと思う人でも、Reactに組み込みで存在していると分かればじゃあキューイングしようとなるかもしれません。だからこそ、技術者は知識を付ける必要があるのです。

ぜひ今後の開発にuseActionStateの知識を活かしてください。

この先の応用としては、カートに追加APIが複数追加をサポートしていたときの最適化とか、そもそもエラーハンドリングどうするの等あるのですが、その話はまた追い追いしたいと思います。