---
title: "まずはSuspenseを試してみよう"
---

では、ここからは手を動かしていきましょう。本書のコードは以下のリポジトリで参照することができます。Viteを使用しているので、`npm install`したら`npm run dev`で開発サーバーを起動してコードを動かしましょう。

https://github.com/uhyo/react-suspense-handson

初期状態は`master`ブランチにあります。ここからスタートしましょう。この章の内容を反映したブランチは`chapter/try-suspense`です。

# 初期状態の確認

初期状態では、何の変哲もないReactアプリです。`App.tsx`内の以下のコードにより画面に「React App!」と表示されているはずです。

```tsx
<div className="text-center">
  <h1 className="text-2xl">React App!</h1>
</div>
```

# とにかくコンポーネントをサスペンドさせてみる

何はともあれ、まずはコンポーネントをサスペンドさせてみましょう。

コンポーネントをサスペンドさせるには、Promiseをthrowすればいいのでしたね。ということで、常に「1秒後に解決されるPromise」を投げるコンポーネントを作ってみましょう。

```tsx
function sleep(ms: number) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

const AlwaysSuspend: React.VFC = () => {
  throw sleep(1000);
};
```

これにより、`AlwaysSuspend`をレンダリングしようとすると常にサスペンドするはずです。

では、このコンポーネントをAppに組み込んでレンダリングし、サスペンドの挙動を確認してみましょう。

```diff tsx
 <div className="text-center">
   <h1 className="text-2xl">React App!</h1>
+  <AlwaysSuspend />
 </div>
```

結果は、**画面が真っ白になる**です。コンソールには次のエラーが表示されているはずです。

> Uncaught Error: AlwaysSuspend suspended while rendering, but no fallback UI was specified.
>
> Add a `<Suspense fallback=...>` component higher in the tree to provide a loading indicator or placeholder to display.

よくよく考えればこのようなエラーが発生するのは当たり前です。サスペンドというのはコンポーネントがレンダリングできない状態です。そもそも関数コンポーネントはその返り値がレンダリングされるのに、`AlwaysSuspend`は何も返していないですね。ということで、この状態では`AlwaysSuspend`のレンダリング結果が存在しない状態なので、Reactはレンダリングを完遂できないのです。

ちなみに、**サスペンドが発生したらその部分だけでなく周りも巻き込んで表示できなくなる**点に留意しましょう。「`AlwaysSuspend`部分だけ何も表示されない」のような挙動ではなく、「`App`全体が表示できない」という挙動になるのです。これはReactが提供する一貫性保証の一部であり、ある瞬間にレンダリングされたコンポーネントツリーが部分的に表示されてしまうようなことを防ぐためであると思われます。全部表示できるか、全部表示できないかのどちらかなのです。

ということで、次はこのエラーに対処しましょう。

# Suspenseで囲む

エラーの対処方法は簡単です。エラーメッセージにも書いてある通り、`AlwaysSuspend`を`Suspense`で囲めばよいのです。

```diff tsx
 <div className="text-center">
   <h1 className="text-2xl">React App!</h1>
+  <Suspense fallback={<p>Loading...</p>}>
    <AlwaysSuspend />
+  </Suspense>  
 </div>
```

こうすると画面が復活します。「React App!」の下に「Loading...」が表示されます。`Suspense`コンポーネントを追加したことで、その内部のサスペンドにこのコンポーネントが対処してくれているのです。`Suspense`の内部でサスペンドが発生した場合、`Suspense`内部は相変わらずレンダリングできませんが、そこにフォールバックコンテンツを表示することによってサスペンドの影響を押さえ込んでくれます。

ここから、`Suspense`コンポーネントが実は**サスペンドの境界を定義する**役割を持っていることがお分かりになるでしょう。先ほど「サスペンドは周りも巻き込んで表示できなくなる」と説明しましたが、その影響は`Suspense`の外に波及しません。`Suspense`の中がサスペンドしてレンダリングできなかったとしても、外側は通常通りにレンダリングできます。

逆に言えば、`Suspense`の中でサスペンドが発生した場合、その`Suspense`の中は全部巻き込まれてレンダリングできなくなります。例えば次のようにしても、「ここは表示される？」は表示されません。

```diff tsx
 <div className="text-center">
   <h1 className="text-2xl">React App!</h1>
   <Suspense fallback={<p>Loading...</p>}>
+   <p>ここは表示される？</p>
    <AlwaysSuspend />
   </Suspense>  
 </div>
```
