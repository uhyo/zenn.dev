---
title: "アクションの概念とuseTransition"
---

[React 19を紹介する公式ブログ記事](https://react.dev/blog/2024/04/25/react-19)において、いの一番の紹介されているのが**アクション** (**Actions**) です。アクションがReact 19を理解する上での鍵となる概念です。実際、React 19で新規に追加されたフックの多くはアクションと関連しています。

ざっくり言えば、アクションは**非同期関数のこと**です。特に、更新系（Web APIへのPOSTなど）の非同期処理を行う関数をアクションと呼びます。

従来、React 18においては、非同期処理を**Suspense**によって取り扱ってきました。Suspenseは非同期処理を待つ間コンポーネントをサスペンド状態にすることができる機能であり、その非同期処理が完了したらコンポーネントのサスペンド状態が解除されて、その非同期処理の結果を用いてコンポーネントが再レンダリングされます。

しかしながら、Suspenseは非同期的なデータの取得に適したAPIであり、更新系の非同期処理には向いていませんでした。React 19で導入されたアクションの概念は、その穴を埋める、更新系の非同期処理を扱うためのAPIとして設計されています。

## アクションをuseTransitionと組み合わせる

アクションの実態は、React 19のAPIが非同期関数を受け取るようになった点にあります。その具体例が **[useTransition](https://react.dev/reference/react/useTransition)** です。

このフックはReact 18から存在していたものですが、React 19で新たに非同期関数に対応しました。

### 従来のuseTransition

従来のuseTransitionは、ステート更新を**トランジション**化するために使われるフックでした。トランジションとなったステート更新は優先度が低いステート更新として扱われ、ステート更新のレンダリング最中に他のステート更新を受け付けたり、ステート更新の最中の表示をレンダリングしたりすることができます。

```tsx:従来のuseTransitionの使用例
const Counter: React.FC = () => {
  const [count, setCount] = useState(0);
  const [isPending, startTransition] = useTransition();

  const handleClick = () => {
    startTransition(() => {
      // このステート更新はトランジション化される
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

`useTransition`のAPIは、このように`startTransition`にコールバック関数を渡すことでステート更新をトランジション化するものです。従来、このコールバック関数は同期関数である必要がありました。

トランジションとなったステート更新による再レンダリングが完了するまで、`isPending`は`true`を返します。このため、`<CountDisplay />`のレンダリングが実はめちゃくちゃ重いという場合に`useTransition`が効果を発揮します。他にも、ステート更新がコンポーネントのサスペンドを誘発した場合、そのサスペンドが解除されるまで`isPending`が`true`になります。

`useTransition`は使いこなせば便利ではありましたが、利用シーンが限定される印象もありました。

### useTransitionのアクション対応

React 19での新機能は、`startTransition`に渡すコールバック関数を非同期関数にすることができる点です。これにより、非同期処理をトランジション化することが可能になりました。

ご想像の通り、非同期処理が完了するまで`isPending`が`true`を返し、その後に再レンダリングが行われます。

```tsx:useTransitionのアクション対応
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

このようにすれば、ボタンを押してからAPIリクエストが完了するまでの間、`isPending`が`true`を返すようになります。

従来は`isPending`のようなステートを自前で管理しなければいけなかったところ、`useTransition`を使うことでReactが管理してくれるようになりました。

## エラー処理

JavaScriptにおいて、非同期処理（Promise）にはエラーが付きまといます。同期関数の場合は呼び出した瞬間にエラーが発生するため、エラー処理は比較的簡単ですが。一方で、非同期関数の場合は話がややこしくなります。

Promiseの管理をReactに任せるということは、Promiseから発生するエラーの対応をReactに任せるということでもあります。

結論としては、非同期関数の中でエラーが発生した場合は、それをキャッチした`useTransition`からエラーが発生します。これにより、レンダリング中に起きたエラーと同様にError Boundaryでキャッチするのが標準的なやり方になります。

ただし、アクションが取り扱うのは更新系であるため、それが失敗したからといって画面がエラー状態に遷移するのは適切でない場合も多いでしょう。その場合は、エラー処理をアクション内で行う（`catch`や`try-catch`を使う）ことになります。

## まとめ

React 19で導入されたアクションの概念は、更新系の非同期処理を扱うのに適しています。`useTransition`は非同期関数を受け取ることができるようになり、非同期処理をトランジション化することが可能になりました。

アクションとは単に非同期関数のことであり、それをアクション対応のReact APIに渡すことでアクションとして扱われると理解するのが良さそうです。

ただ、実はアクション対応のAPIは`useTransition`だけではありません。むしろ、`useTransition`はアクションを扱うための最もプリミティブなAPIという位置づけのようで、他にもアクションを扱うためのAPIがいくつか存在します。次はそれらのAPIについて見ていきましょう。