---
title: "useDeferredValueの初期値"
---

わりと地味な新機能をご紹介します。それは、`useDefferedValue`の第二引数が新たに追加されたことです。

## `useDeferredValue`とは

`useDeferredValue`はReact 18で導入されたフックで、トランジションと関わりがあります。

[公式ドキュメント](https://react.dev/reference/react/useDeferredValue)には次のような例が記載されています。

```jsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  // ...
}
```

`useDeferredValue`は、値の変化をトランジション化して遅延させるものだと解釈できます。まず、何らかの要因によって`query`が変化した場合、`deferredQuery`は即座に変化せず、古い値が返され続けます。このとき、`useDeferredValue`の働きにより、裏でトランジションが発生します。このトランジションは、`deferredQuery`の値を新しい`query`の値に更新するものです。

```text:イメージ
query="" deferredQuery=""

↓ステート更新（非トランジション）↓

query="a" deferredQuery=""

↓ステート更新（トランジション）↓

query="a" deferredQuery="a"
```

つまり、`useDeferredValue`は、ステート更新に付随して自動的にもう1つのステート更新（トランジション）を発生させるものだと理解できます。`deferredQuery`の値が更新されるときは、トランジションということで、`useTransition`の性質と同じことが当てはまります。

例えば、`deferredQuery`の変化に伴う再レンダリングに時間がかかったり、サスペンドしたりすることがあります。その場合にSuspenseのフォールバック表示が出るのを回避したり、あるいは待っている間にさらに`query`が更新されて`deferredQuery`の更新がキャンセルされることもあります。

上の例と同じことを`useDeferredValue`を使わずに行う場合、次のようになります。

```jsx
<input onChange={
  (e) => {
    const value = e.target.value;
    setQuery(value);
    startTransition(() => {
      setDeferredQuery(value);
    })
  }
} value={query} />
```

自前でトランジションの中と外の更新を行う感じですね。これをより宣言的にやってくれるのが`useDeferredValue`です。

## React 19の新機能

React 19では、`useDeferredValue`の第2引数が追加されました。この引数は、`useDeferredValue`の初期値を指定するものです。

```jsx
const deferredQuery = useDeferredValue(query, '');
```

第二引数は、コンポーネントの初回レンダリングの場合に意味を持ちます。初回レンダリングの場合、従来のAPIでは「前回の値」が存在しないため、トランジションが発生しません。

初回レンダリングでもトランジションを発生させたい場合に第2引数を使いましょう。その場合、`deferredQuery`は初回レンダリングではその値が使われます。その後、即座にトランジションが発生して`query`の値に更新されます。

## まとめ

私も正直`useDeferredValue`を活用した経験があまり無いのですが、React 19ではますますトランジションの存在感が増してきますから、`useDeferredValue`をうまく使える機会も増えてくるかもしれません。

`useDeferredValue`の挙動をしっかりと理解し、活用できる場面が無いか常に考えましょう。