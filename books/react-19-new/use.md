---
title: "useフック"
---

`use`は、React 19で新しく追加される特殊なフックです。このフックは特にPromiseの取り扱いを良くするために有用です。

## useの使い方

`use`は直観的に使える面白いAPIとなっています。引数としてPromiseを渡すと、その中身が返り値として返ってくるというものです。本来async関数ではないコンポーネントの中で、`await`のような挙動をしてくれます。

```tsx:useの使い方
const ShowCount: React.FC<{
  count: Promise<number>
}> = ({ count }) => {
  const value: number = use(count);
  return <div>{value}</div>;
};
```

このようなAPIを実現するために、`use`はSuspenseが前提になっています。つまり、`use`に渡されたPromiseが未解決の場合、コンポーネントがサスペンドされます。そして、そのPromiseが解決されたタイミングで再度コンポーネントがレンダリングされます。

ちなみに、Suspenseを前提とする以上、`use`の中身は良く知られた「`Promise`をthrowする」実装になっています。ただし、`use`は生でPromiseをthrowするのではなく、Suspense ExceptionというErrorオブジェクトでラップしてthrowするようです。これは開発者をなるべく混乱させないための配慮でしょう。

`use`に関しては、Reactの内部的な仕組みなど語るべきことが色々とあるのですが、実はすでに別の記事があるので、そちらを参照してください。2022年の記事ですが、React 19でも通用しそうです。

https://zenn.dev/uhyo/articles/react-use-rfc