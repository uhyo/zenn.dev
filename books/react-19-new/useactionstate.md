---
title: "useActionState"
---

React 19では**useActionState**という新しいフックが導入されました。

**useActionState**は、アクションを単に実行するだけでなく、**アクションの結果**を取り扱いたいときに便利なAPIです。

ざっくり言えば、**useActionState**は**useReducer**の**非同期版**のようなフックです。

## useActionStateのAPI

APIはこのようになっています。

```tsx:useActionStateのAPI
// アクションの宣言
const [state, runAction, isPending] = useActionState(
  async (currentState, payload) => {
    // アクション
    // ...
    return newState;
  },
  initialState,
);

const handleClick = () => {
  // アクションの実行
  startTransition(() => {
    runAction(payload);
  });
};
```

目につくのは、APIが全体的にuseReducerと似ていることでしょう。引数としてステートの初期値と、ステートの更新関数を受け取ります。更新関数も、「現在のステート」と「更新に使うペイロード」を受け取り、新しいステートを返すものとなっており、useReducerと同じです。

違うところは、第1引数が非同期関数である点です。それに伴って、返り値に`isPending`が追加されています。これは`useTransition`の`isPending`とだいたい同じで、非同期関数が実行を開始すると`isPending`がtrueになり、実行を終了すると`isPending`が`falseになります。

返り値に含まれる`runAction`というのは、アクションを実行する関数です。`useReducer`で言うところの`dispatch`に相当します。

以上のことから、ステート更新に非同期処理が必要だったり、更新系の処理に紐づいたステートがある場合には、**useActionState**を使うと良いでしょう。

## useActionStateの使い方とポイント

例えば、前回のカウンターコンポーネントは`useState`と`useTransition`を使って書かれていましたが、`useState`の代わりに`useActionState`を使うこともできます。

:::details 前回の実装

```tsx
const Counter: React.FC = () => {
  const [count, setCount] = useState(0);
  const [isPending, startTransition] = useTransition();

  const handleClick = () => {
    startTransition(async () => {
      await fetch('https://api.example.com/increment', {
        method: 'POST',
      });
      setCount(count => count + 1);
    });
  };

  return (
    <div>
      <CountDisplay count={count} />
      <button
        type="button"
        onClick={handleClick}
        disabled={isPending}
      >
        Increment
      </button>
    </div>
  );
}
```

:::

```tsx:useActionStateを用いる実装
const Counter: React.FC = () => {
  const [count, increment, isPending] = useActionState(async (currentCount) => {
    await fetch('https://api.example.com/increment', {
      method: 'POST',
    });
    return currentCount + 1;
  }, 0);

  const handleClick = () => {
    startTransition(() => {
      increment();
    });
  };

  return (
    <div>
      <CountDisplay count={count} />
      <button
        type="button"
        onClick={handleClick}
        disabled={isPending}
      >
        Increment
      </button>
    </div>
  );
}
```

簡単な例ですが、`useActionState`の基本的な使い方が示されています。

注意点として、今回のコードでは`useTransition`は使っていないものの、依然として`startTransition`を使っています。React（18以降）では、`useTransition`を使って`startTransition`を得る方法の他に、Reactから直接`startTransition`をインポートして使うこともできます。このコードではこちらを想定しています。

ここで`startTransition`を使う理由は、**`useActionState`が返した関数はトランジションの内部で呼ばなければならないから**です。このように`useActionState`が作ったアクション（今回は`increment`）を使う際は、まずトランジションを開始して、その中でアクションを呼び出しましょう。

上の例からは`useActionState`が`useReducer`の非同期版であるということがよく分かります。「現在のステート」と「操作」を受け取り、新しいステートを作るというパターンになっていますね。

上の例の場合、`useActionState`で返された関数は`increment()`のように引数無しで呼び出す設計となっており、`setCount(count + 1)`ではありません。これは、このアクションの利用者は「数字を1増やしたい」という意図でアクションを実行するのであって、「具体的な値を決めたい」という意図で使うべきではないということを示しています。

これは特に、`useActionState`のアクションが更新系、つまり副作用を含みがちであることとも関係があります。「アクションが何回実行されたのか」と「ステートの状態」が一致している必要があります。`useState`の場合でも`setCount(count + 1)`ではなく`setCount(prevCount => prevCount + 1)`と書くというプラクティスがありますが、`useActionState`の場合はそれがより重要になります。意識して設計しましょう。

ちなみに、`useActionState`はアクションが並列に実行されないような制御が組み込まれています。`increment`を連打した場合、アクションがキューイングされ、前の実行が完了してから次の実行が開始します。このような制御が`useTransition`には無く、`useActionState`に特有のものです。これは、ステート管理が組み込まれている以上、競合状態を避ける仕組みが必要だからでしょう。

## まとめ

`useActionState`は、非同期アクションを実行するだけでなく、その結果を取り扱うためのフックです。`useReducer`の非同期版といえるAPIを持ちます。