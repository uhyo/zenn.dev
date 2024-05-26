---
title: "useActionState"
---

React 19では**useActionState**という新しいフックが導入されました。

**useActionState**は、アクションを単に実行するだけでなく、**アクションの結果**を取り扱いたいときに便利なAPIです。

ざっくり言えば、アクションを走らせることを担当する**useTransition**と、ステートを管理する**useReducer**を**合体**させたようなフックです。

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
  runAction(payload);
};
```

目につくのは、APIが全体的にuseReducerと似ていることでしょう。引数としてステートの初期値と、ステートの更新関数を受け取ります。更新関数も、「現在のステート」と「更新に使うペイロード」を受け取り、新しいステートを返すものとなっており、useReducerと同じです。

違うところは、第1引数がアクションであり、非同期関数である点です。それに伴って、返り値に`isPending`が追加されています。これは`useTransition`の`isPending`と同じだと思って差支えないでしょう。

返り値に含まれる`runAction`というのは、アクションを実行する関数です。`useReducer`で言うところの`dispatch`に相当します。

以上のことから、ステート更新に非同期処理が必要だったり、更新系の処理に紐づいたステートがある場合には、**useActionState**を使うと良いでしょう。

## useActionStateの使い方とポイント

例えば、前回のカウンターコンポーネントは`useState`と`useTransition`を使って書かれていましたが、`useActionState`を使うと1つにまとめられます。

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
    increment();
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

簡単な例ですが、`useActionState`のうまい使い方が示されています。

特に、**`useReducer`と似たフックである**ことは重要です。つまり、「現在のステート」と「操作」を受け取り、新しいステートを作るというパターンを意識すべきということです。

上の例の場合、`useActionState`で返された関数は`increment()`のように引数無しで呼び出す設計となっており、`setCount(count + 1)`ではありません。これは、このアクションの利用者は「数字を1増やしたい」という意図でアクションを実行するのであって、「具体的な値を決めたい」という意図で使うべきではないということを示しています。

これは特に、`useActionState`のアクションが更新系、つまり副作用を含みがちであることとも関係があります。「アクションが何回実行されたのか」と「ステートの状態」が一致している必要があります。`useState`の場合でも`setCount(count + 1)`ではなく`setCount(prevCount => prevCount + 1)`と書くというプラクティスがありますが、`useActionState`の場合はそれがより重要になります。意識して設計しましょう。

ちなみに、`useActionState`はアクションが並列に実行されないような制御が組み込まれています。`increment`を連打した場合、アクションがキューイングされ、前の実行が完了してから次の実行が開始します。このような制御が`useTransition`には無く、`useActionState`に特有のものです。これは、ステート管理が組み込まれている以上、競合状態を避ける仕組みが必要だからでしょう。

## まとめ

`useActionState`は、アクションを実行するだけでなく、その結果を取り扱うためのフックです。`useTransition`と`useReducer`を合体させたようなAPIを持ち、非同期処理をトランジション化することができます。