---
title: "refコールバックのクリーンアップ関数"
---

React 19の新機能において私が地味に注目しているのが、**refコールバックのクリーンアップ関数**です。

refコールバックとは、要素やコンポーネントの`ref`に関数を渡すことができる機能です。

`ref`と言えば、普通行われるのは`useRef`との組み合わせでしょう。`useRef`はコンポーネントの再レンダリング等を経ても同じオブジェクトを保持し続けるため、レンダリングのサイクルを超えて値を保持するのに便利です。注意点もあり、`useRef`の値はレンダリングの最中に参照すべきではなく、`useEffect`やイベントハンドラ等と組み合わせるのが適切です。

## refコールバックの仕様（従来仕様）

一方で、refコールバックはその名のとおり関数であり、要素に対応するDOMノードが引数として渡されます。

```tsx
<div ref={(node) => {
  // nodeにHTMLDivElementが渡される
  console.log(node);
}}>
  ...
</div>
```

さらに、このノードがDOMツリーから除去されるときは、同じ関数が引数を`null`として呼ばれるようになっています。これによって、関数側がDOMの状態を知ることができます。

上の例の場合、このdivがレンダリングされた時点でコンソールに`HTMLDivElement`が表示され、その後このdivがDOMツリーから除去されると、コンソールに`null`が表示されるというわけです。

### refコールバックと再レンダリング

このようにコンポーネントが再レンダリングされる場合はどうでしょうか。

```tsx
const MyComponent: React.FC = () => {
  const [count, setCount] = useState(0);
  return (
    <div ref={(node) => {
      console.log(node);
    }}>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
};
```

この場合、ボタンをクリックするたびに`MyComponent`が再レンダリングされます。すると、実はそのたびにrefコールバックも再度呼ばれるのです。

これは、refコールバックの関数が毎回新しい関数として作られるからです。この場合、DOMノード（`HTMLDivElement`）の側は変わっていないのですが、ref関数側が変わったため、Reactランタイムは新しい関数に対してDOMノードを同期してあげる必要性を認識します。この場合、古いrefコールバック関数に`null`を渡したあと、新しいrefコールバック関数にDOMノードを渡すことになります。

コードを次のように変えて、どのrefコールバック関数が呼び出されたのか分かるようにしてみましょう。

```tsx
const MyComponent: React.FC = () => {
  const [count, setCount] = useState(0);
  return (
    <div ref={(node) => {
      console.log(count, node);
    }}>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
};
```

これで何回かボタンを押すと、次のようなコンソールログが観察できるはずです。

```
0 <div>...</div>
  (ボタンを押す)
0 null
1 <div>...</div>
  (ボタンを押す)
1 null
2 <div>...</div>
```

この機構により、常に1つのrefコールバック関数がそのDOMノードを管理している（前のコールバック関数はnullを渡されて役目を終える）ことが保証されています。

このため、refコールバック関数を`useCallback`でメモ化してあげた場合は、再レンダリングしてもDOMノードもrefコールバックも変化していないため再呼び出しは行われません。

```tsx:refコールバックをメモ化する例
const MyComponent: React.FC = () => {
  const [count, setCount] = useState(0);
  const refCallback = useCallback((node) => {
    console.log(node);
  }, []);
  return (
    <div ref={refCallback}>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
};
```

この場合は、何らかの要因でDOMノードが変わった場合、およびアンマウントされた場合にのみrefコールバックが呼ばれることになります。

## refコールバックとuseEffectの類似性と相違点

このrefコールバックの機構は、`useEffect`に似ています。`useEffect`はレンダリングのサイクルに合わせて、必要に応じて前のエフェクトをクリーンアップし、新しいエフェクトを有効化します。同様に、refコールバックも前のコールバックをクリーンアップし、新しいコールバックを有効化する動きになっていることが分かります。

しかし、クリーンアップの方法が異なりました。`useEffect`は「前のエフェクトから発行されたクリーンアップ関数を呼び出す」という方法なのに対して、refコールバックは「前のコールバック関数に`null`を渡す」という方法を取っています。

なぜ方法が異なるのかはよく分かりませんが、refコールバックの方法はやや不便です。`null`が渡された場合、どのDOMノードがクリーンアップ対象なのかは自分で覚えておく必要があるのです。

```tsx:自分で覚えておく例
const MyComponent: React.FC = () => {
  const [count, setCount] = useState(0);
  const refCallback = useMemo(() => {
    let memory: HTMLDivElement | null = null;
    return (node: HTMLDivElement | null) => {
      if (node !== null) {
        // 有効化
        memory = node;
        console.log('activation', node);
      } else {
        // クリーンアップ
        console.log('cleanup', memory);
      }
    }
  }, []);
  return (
    <div ref={refCallback}>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
};
```

これはやや不便ですね。ということで、React 19では`useEffect`と同様の仕組みがサポートされました。

## React 19におけるrefコールバックのクリーンアップ関数

React 19では、refコールバックがクリーンアップ関数を返すことができるようになりました。

```tsx:refコールバックのクリーンアップ関数
const MyComponent: React.FC = () => {
  const [count, setCount] = useState(0);
  const refCallback = useCallback((node: HTMLDivElement) => {
    console.log('activation', node);
    return () => {
      console.log('cleanup', node);
    };
  }, []);
  return (
    <div ref={refCallback}>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
};
```

これなら自分でnodeを覚えておく必要がなく、`useEffect`と同じ考え方が通用するので便利ですね。

ちなみに、クリーンアップ関数を返した場合は、refコールバックがnullで呼ばれることは無くなります。refコールバックが関数を返すことによって、従来型（`null`で呼ばれる）をオプトアウトして新しい型（クリーンアップ関数を返す）を選択することができるようになったということです。

## refコールバックとuseEffectの使い分け

React 19ではrefコールバック関数の使い勝手が向上したわけですが、前述のようにrefコールバック関数は`useEffect`と似ているところもあります。そうなると、どちらを使えばいいのか迷ってしまいそうですね。

ここで私が指摘しておきたいのは、**`useEffect`でDOMノードを扱う場合の罠**についてです。

特に、自分のコンポーネント内でレンダリングしたDOMノードを`useRef`で取得し、それを`useEffect`で扱う場合には注意が必要です。

具体的には、何らかの要因でDOMノードが変わった場合、`useEffect`の依存配列の指定によってはDOMノードの変化に反応できないかもしれません。

例えば、React 18までは`onTransitionStart`などのイベントのサポートが無かったため、このイベントを使用するためには自分でDOM操作をする必要がありました。

```tsx:useEffectでDOM操作をする例
const MyComponent: React.FC = () => {
  const ref = useRef<HTMLDivElement>(null);
  useEffect(() => {
    if (ref.current) {
      const node = ref.current;
      const controller = new AbortController();
      node.addEventListener('transitionstart', () => {
        console.log('transition start');
      }, {
        signal: controller.signal,
      });

      return () => {
        controller.abort();
      };
    }
  }, []);
  return (
    <div>
      <div ref={ref}>...</div>
    </div>
  );
};
```

これは、このコンポーネントの場合はうまく動きますが、このコンポーネントをちょっと変えると問題が起こります。

```tsx:問題のあるコンポーネント
const MyComponent: React.FC = () => {
  const [isShown, setIsShown] = useState(false);
  const ref = useRef<HTMLDivElement>(null);
  useEffect(() => {
    if (ref.current) {
      const node = ref.current;
      const controller = new AbortController();
      node.addEventListener('transitionstart', () => {
        console.log('transition start');
      }, {
        signal: controller.signal,
      });

      return () => {
        controller.abort();
      };
    }
  }, []);
  return (
    <div>
      {isShown ? <div ref={ref}>...</div> : null}
      <button onClick={() => setIsShown(s => !s)}>Toggle</button>
    </div>
  );
};
```

このコンポーネントは、ボタンを押すたびに`isShown`がトグルされるため、`ref`を持つ`<div>`要素がマウントされたりアンマウントされたりします。この場合、`useEffect`の依存配列が`[]`であるため、`useEffect`は最初のレンダリング時に1回だけ呼ばれ、その後は再度呼ばれません。そのため、`transitionstart`イベントを与えられたDOMノードが消されたときにクリーンアップされませんし、次のDOMノードが追加されたときにも`transitionstart`イベントが追加されません。

この場合は、`useEffect`の依存配列に`[isShown]`を指定することで解決できますが、エフェクト内で参照されない`isShown`を依存配列に指定するのは良くありません。

このような場合は、そもそも`useEffect`の依存配列を渡さずに毎レンダリングごとにイベントを発火させるのも一つの方法ですが、この方法が取れない場合もあります。`ref`を子コンポーネントに受け渡していて子コンポーネントが勝手に再レンダリングしている場合などです。そのため、実はこのように、**自分がレンダリングしたDOMノードを操作する場合に`useEffect`は適していません**。

そこで、refコールバックを使えば問題をきれいに解決できます。

```tsx:refコールバックを使う例
const MyComponent: React.FC = () => {
  const [isShown, setIsShown] = useState(false);
  const refCallback = useCallback((node: HTMLDivElement) => {
    const controller = new AbortController();
    node.addEventListener('transitionstart', () => {
      console.log('transition start');
    }, {
      signal: controller.signal,
    });

    return () => {
      controller.abort();
    };
  }, []);
  return (
    <div>
      {isShown ? <div ref={refCallback}>...</div> : null}
      <button onClick={() => setIsShown(s => !s)}>Toggle</button>
    </div>
  );
};
```

ReactはrefコールバックとDOMノードの関係を常に最新の状態に保ってくれます。refコールバック側かDOMノード側かのどちらかが変化すれば、古いrefコールバックはクリーンアップされ、新しいほうが有効化されます。

## まとめ

refコールバックは`useEffect`では補いきれないユースケースを持っています。そのrefコールバックがより使いやすくなったのはとても嬉しいですね。
