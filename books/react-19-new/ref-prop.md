---
title: "refをpropに渡すことができる改善"
---

React 19では、`ref`をpropに渡すことができるようになりました。逆に言えば、従来は渡せなかったということです。

```jsx:React 18のrefの挙動
// コンポーネントにrefを渡しても……
<MyComponent ref={myRef} />

// 受け取れない！
const MyComponent = ({ ref }) => {
  // refはundefinedになってしまう
  return <div ref={ref}>Hello</div>;
};
```

従来は、コンポーネントの`ref`は特別扱いされていて、propとして受け取ることはできませんでした。そのため、`forwardRef`という特別な関数を使って、`ref`を受け取ることができるようにしていました。

```jsx:forwardRefの使い方
<MyComponent ref={myRef} />

const MyComponent = React.forwardRef((props, ref) => {
  // refを受け取れる！
  return <div ref={ref}>Hello</div>;
});
```

しかし、React 19では、`ref`をpropとして受け取ることができるようになりました。そのため、`forwardRef`を使う必要がなくなりました。

```jsx:React 19のrefの挙動
<MyComponent ref={myRef} />

const MyComponent = ({ ref }) => {
  // refを受け取れる！
  return <div ref={ref}>Hello</div>;
};
```

この改善により、将来のバージョンでは`forwardRef`が非推奨になることが予告されています。

:::message
この説明は関数コンポーネント限定です。クラスコンポーネントでは元々「refにそのクラスのインスタンスが入る」という仕様があり、これはReact 19でも変わりません。そのため、コンポーネントの中にpropsが伝わることはありません。
:::

## まとめ

従来から、`ref`以外の名前で受け取れば同じことはできていました。しかし、`ref`という名前のほうが直観的なこともあるでしょう。

外から`ref`を渡せるコンポーネントを作る際には活用しましょう。
