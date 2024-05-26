---
title: "useOptimistic"
---

アクションに関わる最後の新フックとして、**useOptimistic**を紹介します。名前から想像されるように、**楽観的更新**を行うためのフックです。

ここでの楽観的更新とは、非同期処理を開始する際に、処理成功後の状態をもうUIに反映してしまうことです。これにより、ユーザーが操作した結果を即座にフィードバックすることができます。ただし、処理が失敗した場合は、その状態を元に戻す必要があります。

楽観的更新は快適なUIのために重要であり、ReactもUIのためのライブラリである以上、楽観的更新が必要になる場面は多いでしょう。そのため、React 19では**アクション**と絡める形でこのフックが導入されました。

楽観的更新が必要なのは非同期な更新系の場合であり、それはまさにアクションの領域です。そのため、`useOptimistic`とアクションに関係があるのは自然なことですね。

## useOptimisticのAPI

APIはこのようになっています。

```tsx:useOptimisticのAPI
const [displayState, addOptimistic] = useOptimistic(
  originalState,
  (currentState, optimisticValue) => {
    // 楽観的更新の処理
    // ...
    return displayState;
  },
);
```

`displayState`は楽観的更新を反映した値であり、`addOptimistic`は楽観的更新を行う関数です。

通常の場合`displayState`は`originalState`と同じになります。パススルーというやつですね。楽観的更新を行うためには、アクションの実行中に`addOptimistic`を呼び出します。そうすることで楽観的更新状態になります。

楽観的更新が行われている場合は、`displayState`は`useOptimistic`に渡された関数によって計算されます。`currentState`は現在の値、`optimisticValue`は`addOptimistic`に渡された引数です。ステートを計算する関数ですから、この関数は当然純粋関数である（副作用がない）ことが求められます。

実際の使い方を見れば、もう少し理解が深まるでしょう。

## useOptimisticの使い方

例えば、少し前の`useActionState`の例に楽観的更新を追加してみましょう。

```tsx:前回の実装
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

この例では、カウンターを増やすアクションが実装されています。今の実装では、ボタンを押した後にAPIリクエストが成功するまでカウンターが増えません。これを楽観的更新に変更してみましょう。

```tsx:useOptimisticを用いる実装
const Counter: React.FC = () => {
  const [count, increment, isPending] = useActionState(async (currentCount) => {
    addOptimistic(1);
    await fetch('https://api.example.com/increment', {
      method: 'POST',
    });
    return currentCount + 1;
  }, 0);

  const [displayCount, addOptimistic] = useOptimistic(count, (currentCount, optimisticValue) => {
    return currentCount + optimisticValue;
  });

  const handleClick = () => {
    increment();
  };

  return (
    <div>
      <CountDisplay count={displayCount} />
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

`useOptimistic`を使って、`count`の値を楽観的更新に対応させています。具体的には、アクションが呼び出されたら、APIリクエストを送る前に`addOptimistic(1)`を呼び出しています。

これにより楽観的更新が起動します。この場合`addOptimistic`に`1`を渡していますから、楽観的更新の計算時は`currentCount === count`、`optimisticValue === 1`となります。この値を使って新しい`displayCount`を計算しています。

ルールとして、`addOptimistic`は**アクションの中**から呼び出す必要があります。つまり、`useTransition`、`useActionState`、あるいはform要素の`action`などに渡されてアクションと化した関数の中から呼び出す必要があります。

楽観的更新は、アクションの最中（典型的には最初）に開始され、そのアクションが終了するまで有効になります。アクションが完了したということは本当の結果が画面に反映されたということなのでもう楽観的更新は不要です。その辺りの制御も自動でやってくれるという、シンプルながらも便利なフックです。

## なぜ関数で楽観的更新を行うのか

`useOptimistic`のAPIは、単純に「楽観的更新中に表示する値」を指定するようにはなっていません。代わりに、元の値と、追加の情報を受け取って新しい値を作るという、これまた`useReducer`を彷彿とさせるAPIになっています。

この理由は、アクションの最中に`originalState`が変化することもありえるからでしょう。例えば「いいね」を押した場合、楽観的更新ではいいね数が「本来の数+1」になるように制御します。その間に他の要因で`originalState`が変化した場合、画面には「変化後の値+1」が表示されるのが正しいはずです。関数を用いたこのようなAPIにすることで、そのようなケースにも対応できるのでしょう。

アクションが非同期トランジションであり、トランジションは元来優先度の低いステート更新を指すということを思い出しましょう。アクションの最中に他の更新が割り込んできた場合も考慮した造りになっているのです。

## まとめ

`useOptimistic`は、アクションの最中に楽観的更新を行うためのフックです。アクションの中から呼び出すことで楽観的更新を開始し、アクションが終了するまで有効です。アクションの結果が画面に反映されると、楽観的更新は自動的に終了します。