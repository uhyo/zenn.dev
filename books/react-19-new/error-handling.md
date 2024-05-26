---
title: "エラーハンドリングの改善"
---

React 19では、エラーハンドリングの挙動の改善が行われました。詳しくは[公式ブログ](https://react.dev/blog/2024/04/25/react-19)を見ていただくのがよいですが、ここでは特に嬉しい改善を紹介します。

APIとしては、react-domの`createRoot`や`hydrateRoot`に対する改善となります。

## 従来つらかった点

従来つらかった点は、Error Boundaryがエラーを正常にキャッチした場合でも、コンソールに`console.error`でエラーが出力されてしまうことです。

Error Boundaryでエラーハンドリングを組んでいるのにエラーが出てしまうことは、開発者に対する負のインセンティブになります。

## React 19の改善

`createRoot`に新たに追加された`onCaughtError`オプションを使うことで、`console.error`でエラーが出力されるというデフォルトの挙動を上書きすることができます。

```tsx
ReactDOM.createRoot(document.getElementById("root")!, {
  onCaughtError: (error) => {
    // infoにできた！（無視してもいい）
    console.info("caught", error);
  },
}).render(...);
```

とても嬉しいですね。

また、対となる`onUncaughtError`も追加されました。これはError Boundaryがキャッチしなかった場合ですね。

