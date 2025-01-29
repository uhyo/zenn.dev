---
title: "<form>とアクション"
---

アクションの解説はまだ続きます。React 19では、form要素にもアクションの対応が組み込まれました。

## form要素のアクション

React 19では、formの`action`属性にアクションを指定することができます。これにより、フォームの送信時にアクションが実行されるようになります。従来`onSubmit`イベントを使っていた場面で、`action`属性を使うことができるイメージです。

「アクションを指定する」ということは、つまり非同期関数を指定するということです。

```tsx:form要素のアクション
const Counter: React.FC = () => {
  const [count, setCount] = useState(0);

  return (
    <form action={
      async () => {
        await fetch('https://api.example.com/increment', {
          method: 'POST',
          body: JSON.stringify({ count: count + 1 }),
        });
        setCount(count + 1);
      }
    }>
      <CountDisplay count={count} />
      <button
        type="submit"
      >
        Increment
      </button>
    </form>
  );
}
```

こうすることで、フォームの送信時に非同期処理を実行することができます。一般に、フォームを含むUIでは、ボタンに`onClick`イベントなどを設定するよりも「送信」をトリガーに動くようにしたほうが適切です。このような場面では`action`を積極的に使っていきたいです。

エラー処理に関してもこれまでと同様で、アクションがエラーを発生させた場合には`<form>`でエラーが発生した扱いとなり、ErrorBoundaryでキャッチできます。

## useActionStateと併用する

上の例は`action`の使い方を示していますが、これまでの例に比べて雑な気がします。例えば`isPending`がありません。また、前回`setCount(count + 1)`は良くないよと言っていたわりに、上の例はそうなっています。

これを解消するためには、前回紹介した`useActionState`と組み合わせるのが有効です。[useActionStateのドキュメント](https://react.dev/reference/react/useActionState)を見ても、formとの組み合わせが特に推奨されています。

```tsx:useActionStateを用いる実装
const Counter: React.FC = () => {
  const [count, increment, isPending] = useActionState(async (currentCount) => {
    await fetch('https://api.example.com/increment', {
      method: 'POST',
    });
    return currentCount + 1;
  }, 0);

  return (
    <form action={increment}>
      <CountDisplay count={count} />
      <button
        type="submit"
        disabled={isPending}
      >
        Increment
      </button>
    </form>
  );
}
```

前回、`useActionState`が返した関数（`increment`）は`startTransition`でラップして呼び出す必要があると説明しましたが、このように`<form>`の`action`に渡す場合はその必要がありません。なぜなら、`<form>`の`action`は自動的にトランジションとなるからです。つまり、このように`action`属性を介して使う場合は`startTransition`は必要ありません。

そのため、最も単純な形ではこのように`increment`を直接`action`属性に渡しても問題ありません。

つまるところ、Reactではアクションを呼び出す場合はトランジションを使う必要があり、その方法が「`startTransition`を使う」か「`<form>`の`action`属性を使う」の2種類あるということです。

ただ、`isPending`が欲しいからという理由だけで`useActionState`を使うのが大げさな場合もあるでしょう。`useActionState`には自動的にステート管理が付属します。ステート管理は不要という場合には別のフックを使うのが有効です。

## useFormStatusと併用する

ここで新しいフックを紹介します。`useFormStatus`は、`<form>`との併用が前提となっているため、`react`ではなく`react-dom`からエクスポートされています。このようなフックは今のところ`useFormStatus`だけです。

`useFormStatus`は、`<form>`の中にレンダリングされるコンポーネントで使用することができ、その`<form>`の状態を取得することができます。これは、`<form>`がContextのような役割をするものと説明されています。

実は、`isPending`相当のものは、`useActionState`を使わなくても`<form>`自体に組み込まれているのです。その状態を取得する手段が`useFormStatus`です。

```tsx:useFormStatusの使用例
const IncrementButton = () => {
  const isPending = useFormStatus();

  return (
    <button
      type="submit"
      disabled={isPending}
    >
      Increment
    </button>
  );
};

const Counter: React.FC = () => {
  const [count, increment] = useState(0);

  return (
    <form action={
      async () => {
        await fetch('https://api.example.com/increment', {
          method: 'POST',
        });
        increment(count + 1);
      }
    }>
      <CountDisplay count={count} />
      <IncrementButton />
    </form>
  );
};
```

これにより、`useActionState`が無くても`isPending`の挙動をさせることができました。

## FormDataのサポート

基本的な`<form action>`の使い方は以上の通りなのですが、React 19のフォームはもう少し奥が深いものになっています。

フォームコントロールの実装には、制御コンポーネント (controlled component) と非制御コンポーネント (uncontrolled component) の2つの方法があることが知られています。React 19では、非制御コンポーネントのサポートが充実しました。

非制御コンポーネントは、入力内容をReactのステートで管理せず、DOMに任せるやり方です。フォームの送信時などデータが必要な場合はDOMのAPIを使ってデータを取得します。DOMにおいて今どきのやり方は`FormData`を使うことです。

```ts:FormDataの使用例
const formElement = document.querySelector('#my-form');

// そのformに入力されたデータをFormDataオブジェクトとして取得
const formData = new FormData(formElement);
```

React 19では、これをReact側でやってくれるようになりました。`<form>`の`action`属性に渡された関数には、`FormData`オブジェクトが引数として渡されます。これにより、何もしなくてもフォームのデータを取得することができます。

```tsx:FormDataの使用例
const Counter: React.FC = () => {
  const [count, setCount] = useState(0);

  return (
    <form action={
      async (formData) => {
        const command = formData.get('command');
        await fetch('https://api.example.com/increment', {
          method: 'POST',
          body: JSON.stringify({
            // Web開発あるある: incrementというパスなのにcountが指定できるAPIに変わる
            count: command === 'Increment' ? count + 1 : count - 1,
          }),
        });
        setCount(count + 1);
      }
    }>
      <CountDisplay count={count} />
      <button
        type="submit"
        name="command"
      >
        Increment
      </button>
      <button
        type="submit"
        name="command"
      >
        Decrement
      </button>
    </form>
  );
}
```

この例では渡された`FormData`を使って、IncrementとDecrementのどちらのボタンが押されたかを判定しています。

### useFormStatusとFormData

実は、`useFormStatus`からも`FormData`を取得することができます。

```tsx
const { isPending, data } = useFormStatus();
```

ただし、`data`が得られるのはフォームの送信中のみ、つまり`isPending`が`true`のときだけです。現在のフォームの入力データをリアルタイムに取得できるAPIではないので注意してください。`isPending`が`true`の場合の描画をするときに`FormData`のデータを使いたいという需要のために用意されています。

他にも`method`と`action`も`useFormStatus`から取得できます。

## formとServer Actionsとの関係

特にNext.jsを使っている方は、React 19が出る以前から**Server Actions**についてご存じだったかもしれません。これは、React Server Components (RSC) の利用を前提とし、Reactのソースコードの中にサーバー側で実行される関数を定義し、あたかもクライアント側から直接サーバー側の関数を呼び出しているかのように見える技術です。

form要素の`action`属性は、Server Actionsと深いかかわりがあります。これまで解説した内容はRSCと関係なく利用できますが、Server Actionsを使うことでform要素がさらに強化されます。この本ではフレームワーク周りまで踏み込むつもりは無いのですが、formについてはさすがに触れておきたいと思います。

基本的な使い方としては、Server Actionsもこの本で紹介してきたアクションの一種であり（非同期関数ですからね）、`action`属性に渡すことができます。

```tsx:Server Actionsを使う
async function increment() {
  "use server";
  await db.sql`UPDATE count SET count = count + 1`;
}

const Counter: React.FC = () => {
  return (
    <form action={increment}>
      <button
        type="submit"
      >
        Increment
      </button>
    </form>
  );
}
```

このようにすると、フレームワークやReactのランタイムが裏でよしなにやってくれて、サーバー側で`increment`関数が実行されます。これにより、クライアント側からサーバー側の関数を呼び出すことができます。

問題は、**ハイドレーション**がまだの場合です。というのも、サーバーまで巻き込んだフレームワークを使う場合、SSRが行われるのが普通です。SSRは、サーバーで生成されたHTMLをクライアントに送ることで画面の初期表示を早くする技術です。この時に行われるのがハイドレーションです。ハイドレーションは、クライアント側でもReactの描画を実行することで、イベントハンドラなどクライアント側の挙動を有効化することです。SSRされたHTMLがハイドレーションされるまでの一瞬の間はユーザーの操作が効かないというのがハイドレーションの問題点であり、その問題を軽減するための工夫が行われています。

formの`action`にServer Actionを指定した場合でも、ハイドレーション済の状態では、実際の送信（submitイベント）はキャンセルしてJSでRPCを処理してくれます。しかし、ハイドレーション前の状態では、フォームが送信されてしまいます。React 19ではこの場合ですらサポートしようとしています。なお、これは公式ドキュメントなどでProgressive Enhancementと呼ばれていますが、筆者はこの言葉の使い方が嫌いなのでこの言葉への言及はこれ以降避けます。

サーバーを担当するフレームワークがこの場合にformから送信されたリクエストを受け付けて、どのServer Actionが実行されたのかを判定し、実際にそれを実行してくれます。そして、実行後の状態をレンダリングすることにより、結果的にアプリが想定通りの挙動をしたように見えるのです。

formの`action`に渡されたアクションが引数として`FormData`を受け取るのも、このことと関係があると言えます。ハイドレーション前の状態でのフォーム送信をサポートする場合、HTMLの仕様に従ってサーバー側に送られたフォームのデータが、唯一利用可能な入力となります。このため、formの`action`は最初から`FormData`を受け取るアクションに限定されているのでしょう。

### formとuseActionStateの関係

Server Actionの機能は限定されています。サーバー側で実行されるため、クライアント側の状態に関与することはできません。少し上で、formの`action`に渡した非同期関数の中で`setCount`している例がありましたが、あれはServer Actionではないからできることです。

しかし、Server Actionから返り値を得ることはできるはずです。実は、このことと、前述のハイドレーション前の送信のサポートを両立するという役割を担っているのが`useActionState`です。

```tsx:useActionStateとServer Actionの組み合わせ
async function increment() {
  "use server";
  const result = await db.sql`UPDATE count SET count = count + 1 RETURNING count`;
  return result.count;
}

const Counter: React.FC = () => {
  const [count, runIncrement, isPending] = useActionState(increment, 0);

  return (
    <form action={runIncrement}>
      <CountDisplay count={count} />
      <button
        type="submit"
        disabled={isPending}
      >
        Increment
      </button>
    </form>
  );
};
```

このように`useActionState`でServer Actionをラップして、しかもそれをformの`action`に渡します。そうすると、あとはフレームワーク側がうまくやってくれます。

ハイドレーション前にフォームが送信された場合は前述のようにサーバー側でServer Actionが実行されます。その結果は、サーバーからのレスポンスで使用されます。つまり、Server Actionの結果が対応する`useActionState`のステートに最初から入った状態で、レスポンスが作られます。

この仕組みによって、クライアント側でJavaScriptを介さずにフォームの送信を行ったとしても、その結果がクライアント側のステートに反映されるようになります。これにより、ハイドレーション前のフォーム送信がサポートされることになります。

## まとめ

React 19では、form要素にアクションを指定することができるようになりました。これにより、フォームの送信時に行う非同期処理をきれいに定義することができるようになりました。

また、Server Actionsと組み合わせてJavaScriptを介さない状態でもアプリが動作できるようにするための必須パーツという側面もあります。
