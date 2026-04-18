---
title: "Async React時代の宣言的UI 2: トランジション対応のuseDebouncedフックを作る"
emoji: "⛹️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

皆さんこんにちは。以下の記事では、Async React時代の宣言的UIとして、**デバウンス**を`useDeferredValue`で代替する方法を示しました。

https://zenn.dev/uhyo/articles/async-react-debounce

記事の末尾で「実際には、ネットワークアクセスをデバウンスしている場合とか応用形もあるのですが」と述べたので、今回はネットワークアクセスを含む場合について考えたいと思います。

今回の記事に登場するコードは以下のStackBlitzで実際に動作を確認できます。

@[stackblitz](https://stackblitz.com/edit/vitejs-vite-ldcezusd?embed=1&file=src%2FApp.tsx)

## 今回の要件

前回の記事では、ユーザーが入力すると、フロントエンドで検索結果の計算（フィルタリング）が走り、その結果が表示される例を考えました。この例ではネットワークアクセスが必要ないので、ユーザーの入力を何ミリ秒デバウンスするとか、そういうことを考える必要がありませんでした。そのため、`useDeferredValue`を使うだけで自動的に最適化した体験が提供できました。

今回は、検索結果がバックエンドから得られるという設定にしましょう。そうなると、ネットワークアクセス（いわゆるAPI呼び出し）が必要になります。

この場合、ユーザーが1文字入力するごとにAPI呼び出しが発火するのは困ります。そこで、**API呼び出しを間引くためにユーザーの入力をデバウンスする**という要件が生まれます。

具体的には、例えば間隔を500ミリ秒に設定したとすると、「ユーザーが最後の入力をしてから500ミリ秒経過してからAPI呼び出しを行う」という要件になります。

## ベース実装

```tsx
import { Suspense, use, useState } from 'react';
import { memoizeOne } from './utils/memoizeOne';
import { api } from './api';
import { useDebouncedValue } from './useDebouncedValue';

// api: (searchText: string) => Promise<string[]>
const cachedApi = memoizeOne(api);

function App() {
  const [text, setText] = useState('');

  const searchText = useDebouncedValue(text, {
    intervalMs: 500,
  });

  const searchResult = cachedApi(searchText);

  return (
    <section id="center">
      <h1>useDebounced test</h1>
      <p>
        <input
          value={text}
          onChange={(event) => {
            setText(event.currentTarget.value);
          }}
        />
      </p>
      <div className="result-area">
        <Suspense fallback={<p>Loading...</p>}>
          <SearchResult searchResult={searchResult} />
        </Suspense>
      </div>
    </section>
  );
}

const SearchResult: React.FC<{
  searchResult: Promise<string[]>;
}> = ({ searchResult }) => {
  const list = use(searchResult);
  return (
    <ul>
      {list.map((text) => (
        <li key={text}>{text}</li>
      ))}
    </ul>
  );
};
```

データ取得はもちろんSuspenseベースですが、簡易的に`memoizeOne`で`api`をラップした`cachedApi`関数を用意しています。これはさすがに簡易的すぎますが、実際は適当なデータ取得ライブラリを使っていると想定してください。

ポイントは以下のところですね。

```ts
const searchText = useDebouncedValue(text, {
  intervalMs: 500,
});
```

`text`はユーザーの入力をリアルタイムに反映していますが、`useDebouncedValue`を通すことで、`searchText`はユーザーの入力を500ミリ秒デバウンスした値になるという想定です。これを使うことで、API呼び出しも自動的にデバウンスされることになります。

**この`useDebouncedValue`をどう実装するか**がこの記事の主題です。タイトルにあるとおり、この記事では**トランジション対応**の`useDebouncedValue`を紹介します。

## 従来の実装

従来の（ここでは、Async Reactの概念を採り入れる前の）教科書的な実装はだいたいこんな感じでしょう。

```tsx
interface UseDebouncedValueOptions {
  intervalMs: number;
}

function useDebouncedValue<T>(latestValue: T, options: UseDebouncedValueOptions): T {
  const [debouncedValue, setDebouncedValue] = useState(latestValue);

  const runTimeout = useEffectEvent(
    (value: T) => {
      if (value === debouncedValue) return;
      const timeoutId = setTimeout(() => {
        setDebouncedValue(value);
      }, options.intervalMs);

      return () => {
        clearTimeout(timeoutId);
      };
    },
  );

  useEffect(() => {
    return runTimeout(latestValue);
  }, [latestValue]);

  return debouncedValue;
}
```

この実装は、ユーザーの入力が変わるたびに`setTimeout`でタイマーをセットし、前のタイマーはクリアするという典型的なデバウンスの実装です。`useEffect`を`value`の変化に反応させるのはどうなのとちょっと思いますが、`value !== debouncedValue`なステートがマウントされる場合はタイマーが付随すると解釈すると、Reactのベストプラクティスの範囲内と解釈できそうです。

## トランジションに対応させる

今回紹介する実装の肝は**トランジション対応**です。つまり、ちょうど`useDeferredValue`の返り値がトランジションで更新されるのと同様に、`useDebouncedValue`の返り値が新しくなるときも、トランジションで更新されるようにしたいのです。

自分で考えてみたい読者は、ここで一旦考えてみてください。どうすればトランジション対応の`useDebouncedValue`が実装できるでしょうか？

筆者が考える答えは以下のような実装です。

```tsx
import { useState, useEffect, startTransition as globalStartTransition, type TransitionStartFunction } from 'react';

interface UseDebouncedValuesOptions {
  intervalMs: number;
  startTransition?: TransitionStartFunction;
}

export function useDebouncedValue<T>(
  latestValue: T,
  {
    intervalMs,
    startTransition = globalStartTransition,
  }: UseDebouncedValuesOptions
): T {
  const [debouncedValue, setDebouncedValue] = useState(latestValue);

  const runTransition = useEffectEvent((value: T) => {
    if (value === debouncedValue) return;
    const controller = new AbortController();
    startTransition(async () => {
      const shouldUpdate = await sleep(intervalMs, controller.signal).then(
        () => true,
        (error) => {
          if (error.name === 'AbortError') return false;
          throw error;
        }
      );
      if (!shouldUpdate) return;
      startTransition(() => {
        setDebouncedValue(value);
      });
    });
    return () => {
      controller.abort();
    };
  });

  useEffect(() => {
    return runTransition(latestValue);
  }, [latestValue]);

  return debouncedValue;
}
```

トランジションを発生させる方法のひとつは`useTransition`から得られた`startTransition`関数を呼び出すことですが、今回はオプションで`startTransition`関数を受け取るようにしています。この理由は後で説明します。

あとはタイマーとステート更新を`startTransition`でラップしただけです。ただし、ひとつ特筆すべき点があります。

それは、`startTransition`がネストしている点です。

```ts
startTransition(async () => {
  // ...
  startTransition(() => {
    setDebouncedValue(value);
  });
});
```

こうなっている理由は[React公式ドキュメントに説明がある通り](https://ja.react.dev/reference/react/useTransition#react-doesnt-treat-my-state-update-after-await-as-a-transition)、JavaScriptの言語仕様の制約上、`startTransition`のコールバック内で非同期処理を行うと、`await`後に呼び出されるステート更新が`startTransition`の中で呼び出されていると認識されなくなってしまうためです。それを補うためにこのようにネストさせています。

では、外側の`startTransition`は何のためにあるのでしょうか。ここが一番のポイントです。こうすることで、もし`startTransition`が`useTransition`由来だった場合、**対応するisPendingを即座にtrueにする**効果があるからです。

つまり、この`useDebouncedValue`は以下のような動作をします。

1. 初期状態: `latestValue === "", debouncedValue === "", isPending === false`
2. ユーザーが入力: `latestValue === "a", debouncedValue === "", isPending === false`
3. `startTransition`が呼び出される: `latestValue === "a", debouncedValue === "", isPending === true`
4. 500ミリ秒後: `latestValue === "a", debouncedValue === "a", isPending === false`

これは、React 19で`startTransition`が同期関数だけではなく非同期関数も受け取れるようになったことを利用しています。React 18では、`startTransition`は同期的にステート更新をラップする役割だけを持っていました。そのステート更新が行われると同時に`isPending`がtrueになります。

React 19では、`startTransition`は非同期関数も受け取れるようになりました。コールバック関数がPromiseを返した場合、ReactはそのPromiseが解決されるまでトランジションが続いているとみなします。つまり、コールバック関数内で非同期処理を行うと、その非同期処理が完了するまで`isPending`がtrueのままになります。

## 使用側のコード

このトランジション対応版`useDebouncedValue`は、使う側もトランジションに対応させることで真価を発揮します。使用例は次のとおりです。

```tsx
const cachedApi = memoizeOne(api);

function App() {
  const [text, setText] = useState('');
  const [isPending, startSearchTransition] = useTransition();

  const searchText = useDebouncedValue(text, {
    intervalMs: 500,
    startTransition: startSearchTransition,
  });

  const searchResult = cachedApi(searchText);

  return (
    <section id="center">
      <h1>useDebounced test</h1>
      <p>
        <input
          value={text}
          onChange={(event) => {
            setText(event.currentTarget.value);
          }}
        />
      </p>
      <div className={`result-area ${isPending ? 'pending' : ''}`}>
        <Suspense fallback={<p>Loading...</p>}>
          <SearchResult searchResult={searchResult} />
        </Suspense>
      </div>
    </section>
  );
}
```

最初のコードとの変更点は、`useTransition`を呼び出して`isPending`と`startSearchTransition`を得ていることと、`useDebouncedValue`に`startTransition: startSearchTransition`を渡していること、そして結果エリアのクラス名に`isPending`に応じたクラスを追加していることです。

`isPending`の典型的な使用法は、古くなった検索結果をグレーアウトするなどのスタイルを当てることです。これにより、検索結果が古くなっていることをユーザーに示します。

先ほど説明したように、ユーザーが入力すると即座に`isPending`がtrueになります。これは、デバウンスが終了して`searchText`に入力が反映されるまで継続します。

## トランジション対応useDebouncedValueの評価

前回の記事に引き続き、この実装は**Async Reactや宣言的UI**の考え方をよく反映したものになっています。

デバウンスの実装詳細をuseDebouncedValueに隠しつつ、「デバウンス中」という状態のコミュニケーションを、Reactに組み込みのトランジションという仕組みに乗って行っています。これはまさにReact的な宣言的UIを体現していると言えます。

一応、別のAPIとして、`const [debouncedValue, isPending] = useDebouncedValue(...)`のようにすることも考えられます。

```ts:isPendingを返す実装（概略）
function useDebouncedValue<T>(
  latestValue: T,
  {
    intervalMs,
  }: UseDebouncedValuesOptions
): [T, boolean] {
  const [debouncedValue, setDebouncedValue] = useState(latestValue);
  // ...（省略）
  const isPending = latestValue !== debouncedValue;
  return [debouncedValue, isPending];
}
```

それでもだめとは言いませんが、トランジションを設計の基礎に据える考え方からは、`isPending`という値を受け渡すよりは、「どのトランジションに属するのか」という情報を`startTransition`の受け渡しを通じて行うインターフェースのほうがReact的で面白い設計だと筆者は考えています。

例えば、テキスト入力の他にラジオボタンも検索条件に含まれる場合、ラジオボタンのほうはデバウンスが不要ですがトランジションは依然必要です。そういう場合、`startSearchTransition`を両方のステート更新で使うことで、両方のステート更新が同じトランジション（検索条件を更新するトランジション）に属することを示すことができます。

## 実装の惜しい点

ただ、ちょっと惜しい点があります。このフックは以下のような動作をすると先ほど説明しました。

1. 初期状態: `latestValue === "", debouncedValue === "", isPending === false`
2. ユーザーが入力: `latestValue === "a", debouncedValue === "", isPending === false`
3. `startTransition`が呼び出される: `latestValue === "a", debouncedValue === "", isPending === true`
4. 500ミリ秒後: `latestValue === "a", debouncedValue === "a", isPending === false`

2と3が分かれているのがちょっと惜しい点です。理想的にはユーザーが入力した瞬間（`latestValue`が変わった瞬間）に同じタイミングで`isPending`もtrueになってほしいところですが、現状の実装では、ユーザーが入力した瞬間はまだ`isPending`はfalseのままで、`useEffect`が走ってから`isPending`がtrueになります。これは、2の状態が実際にマウントされてしまうということです。

これは、Reactではコンポーネントのレンダリング中にトランジションを開始できないため、このAPIでは仕方なさそうだと筆者は考えています。

前述の`isPending`も返すAPIの場合は2の時点でtrueを返すことができるのですが、現在のAPIにも前述のメリットがあるので、一長一短ですね。

## まとめ

今回は、Async React時代の宣言的UIの一例として、トランジション対応の`useDebouncedValue`フックを紹介しました。このフックは、ユーザーの入力をデバウンスしつつ、その状態変化をトランジションに載せることができます。

このように、「トランジション」という共通言語を意識してステート変化を含む設計のときは、常にトランジションについて意識することでよい設計ができるようになるでしょう。