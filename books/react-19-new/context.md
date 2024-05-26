---
title: "<Context.Provider>の非推奨化"
---

React 19のちょっとした変更点として、`Context.Provider`が非推奨化されました（正確に言うと、将来のバージョンで非推奨になると予告されました）。

React 19では、`Context.Provider`ではなく`Context`そのものをProviderとして使うことができます。

```tsx
// これまでの書き方
<Context.Provider value={value}>
  <Child />
</Context.Provider>

// 新しい書き方
<Context value={value}>
  <Child />
</Context>
```

改めて考えてみれば、この変更は自然なことのように思えます。

Contextの機能が導入された当時、`Context.Provider`は`Context.Consumer`と対になっていました。しかし、`useContext`の登場により、`useContext(Context)`で値を読みだせるようになったため、`Context.Consumer`は役目を終えました。

現在は、`Context.Provider`は使うのに`Context.Consumer`は使わないという非対称的な状態になっていました。今回の変更は、それなら`Context.Provider`も使わなくていいようにしようという方向性なのでしょう。

今回の変更により、供給側は`<Context>`、使用側は`useContext(Context)`となり、再び対称的な感じが戻ってきました。ちなみに、React 19からは`use`を使って`use(Context)`でも可能です。