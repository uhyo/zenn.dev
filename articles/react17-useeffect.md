---
title: "React17におけるuseEffectの破壊的変更を理解する"
emoji: "🚰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "javascript"]
published: true
---

しばらく前、[React 17 RC](https://reactjs.org/blog/2020/08/10/react-v17-rc.html)が発表されました。現行のReact 16に比べて、いくつかの破壊的変更がある一方、新機能が何もないというのが特徴です。Reactチームとしては、新機能が無いとはいえ、破壊的変更も少なくなっておりなるべく16から17へのアップデートを行なってほしいという考えのようです。

この記事では、React 17における破壊的変更のうち、`useEffect`のクリーンアップのタイミングに関する変更を取り上げます（以下は公式サイトから引用）。

> In React 17, the effect cleanup function also runs asynchronously — for example, if the component is unmounting, the cleanup will run _after_ the screen has been updated.
> 
> （筆者による翻訳） React17では、effectクリーンアップ関数も非同期的に実行されるようになります。例えば、コンポーネントがアンマウントされるとき、クリーンアップ関数は画面が更新_された後_に実行されます。

React 16からReact 17以降への移行をスムーズにするために、たとえいますぐReact 17にアップデートするつもりが無いとしても、React 17に適合した書き方をするのが得策です。そこで、この記事では`useEffect`のクリーンアップ関数の実行タイミングについて詳しく学びます。

## `useEffect`の復習

`useEffect`はフックの一種であり、コンポーネントがレンダリングされたときに実行される関数を指定します。例えば次のコンポーネント`Foo`は、レンダリングされるたびに`useEffect`で指定された関数が実行され、コンソールに`foo!`と表示するでしょう。より具体的には、まず最初に`Foo`がレンダリングされたとき（マウントされたとき）に`foo!`と表示し、その後`Foo`が再レンダリングされるたびに`foo!`と表示します。


```tsx
const Foo = () => {
  useEffect(() => {
    console.log("foo!");
  });
  return <p>I am foo</p>;
};
```

`useEffect`に第2引数を渡すことで関数が実行されるタイミングを制御できるのはご存知の通りです（まだご存知でない方は公式ドキュメントなどを通じて学習しましょう）。特に、第2引数に`[]`を指定することによって、コンポーネントがマウントされたときに1回だけ関数を実行することが可能です。

```tsx
const Foo = () => {
  useEffect(() => {
    console.log("Fooがマウントされました！");
  }, []);
  return <p>I am foo</p>;
};
```

### `useEffect`のタイミング

`useEffect`の特徴は、関数が呼ばれるのが**レンダリングが画面に反映された後である**ということです。

これにより、`useEffect`のコールバック関数の中では、DOMを見れば自身によってレンダリングされた物を観察できます（`react-dom`を使用している場合。React Nativeの場合はDOMではありませんが、ここではDOMの場合で話を進めるので適宜読み替えてください）。Reactではrefを用いると生のDOMオブジェクトを取得でき、`useEffect`の関数内ではすでにこれが利用可能な状態になっています。このことは次のような例で確かめられるでしょう。

```tsx
const Foo = () => {
  const pRef = useRef();
  useEffect(() => {
    // "I am foo" と表示される
    console.log(pRef.current.textContent);
  }, []);
  return <p ref={pRef}>I am foo</p>;
};
```

この例では、`Foo`によってレンダリングされた`p`要素のDOMオブジェクトが`pRef.current`に入ります。`useEffect`のコールバック関数内ではこの処理がすでに完了しているので`pRef.current`が利用可能です。上の例では、その`textContent`プロパティ（中のテキストを文字列で取得する）を使用しています。`Foo`によって`<p>I am foo</p>`がレンダリングされたので、`pRef.current.textContent`は`"I am foo"`となっています。

あまり勧められたことではありませんが、`useEffect`からDOMに干渉することもできます。例えば`pRef.current.textContent`を書き換えることもできるでしょう。

```tsx
const Foo = () => {
  const pRef = useRef();
  useEffect(() => {
    pRef.current.textContent = "I am not foo";
  }, []);
  return <p ref={pRef}>I am foo</p>;
};
```

この`Foo`コンポーネントは`<p>I am foo</p>`をJSXによりレンダリングしますが、`useEffect`で`p`の中身を`I am not foo`に書き換えているため、画面には`I am not foo`と表示されます。ただし、`useEffect`のコールバック関数はJSXの内容が画面に反映されてから呼び出される関係上、実際にやってみると一瞬だけ`I am foo`が見えます。

ただし、`useEffect`の代わりに`useLayoutEffect`を使うとこの一瞬の隙を塞ぐことができます。このことから、コールバック関数で画面に影響を与える場合は`useLayoutEffect`が適しています（上の例のように`textContent`を書き換えるようなことは意味がないのでやめるべきですが）。逆に言えば、`useEffect`の処理は画面に影響を与えない場合に適しています（例えばイベントハンドラを付加するとか）。Reactは、`useEffect`の処理（画面に影響を与えないはず）の実行を、レンダリング結果のDOMへの反映および画面への反映より後にすることによって、レンダリング結果が画面に反映されるまでの時間を短くしているのです。

このことは、コールバック関数の中で時間のかかる処理をしてみると分かります。例えば、次の例は1秒間ループし続ける`run1sec`という悪い関数を定義し、`useEffect`のコールバック関数の中で使っています。

```tsx
const Foo = () => {
  const pRef = useRef();
  useEffect(() => {
    run1sec();
    pRef.current.textContent = "I am not foo";
  }, []);
  return <p ref={pRef}>I am foo</p>;
};

function run1sec() {
  const start = performance.now();
  while (performance.now() - start < 1000);
}
```

この状態で`Foo`をレンダリングすると、まず`I am foo`と表示される→1秒後に`I am not foo`になる、という挙動が観察できます。これにより、`useEffect`のコールバック関数がレンダリング結果がDOMに反映され、画面にもされた後に実行されていることがよく分かります。

一方、`useLayoutEffect`に替えてみましょう。

```tsx
const Foo = () => {
  const pRef = useRef();
  useLayoutEffect(() => {
    run1sec();
    pRef.current.textContent = "I am not foo";
  }, []);
  return <p ref={pRef}>I am foo</p>;
};
```

こうすると、`Foo`をレンダリングしても最初は何も表示されません。1秒後になって`I am not foo`と表示されます。このことから、`I am foo`がDOMに反映後まだ画面に反映されていないタイミングで`useLayoutEffect`のコールバックが実行されていることが分かります。まとめると、`useEffect`や`useLayoutEffect`のコールバック関数の実行のタイミングは以下のように整理できます。

- **useEffect**: レンダリング→レンダリング結果をDOMに反映→DOMを画面に反映→**コールバック関数を実行**
- **useLayoutEffect**: レンダリング→レンダリング結果をDOMに反映→**コールバック関数を実行**→DOMを画面に反映

先の例からも分かるように、`useLayoutEffect`のコールバック関数に時間がかかる場合、レンダリングが画面に反映されるまでの時間に影響します。一方、`useEffect`なら影響しません。このことから、レンダリング結果がなるべく早く画面に反映されるように、なるべく`useLayoutEffect`ではなく`useEffect`を使わなければなりません。

さて、すでに何か勉強した気になったかもしれませんが、これはReact 17と特に関係ない話なのでまだ復習です。

## `useEffect`のクリーンアップ関数 (React 16の場合)

これまで扱ってきた`useEffect`のコールバック関数は、コンポーネントがマウントされたときや再レンダリングされたときに実行されました。それと対になるものとして、**コンポーネントがアンマウントされる直前**、または**次のコールバック関数が呼び出される直前**に呼び出される関数を`useEffect`はサポートしています。これがクリーンアップ関数です。クリーンアップ関数は、コールバック関数が関数を返り値として返すことで指定できます。言い方を変えると、クリーンアップ関数を作るのはコールバック関数であり、これがコールバック関数とクリーンアップ関数を対になるものとしています。

```tsx
const Foo = () => {
  useEffect(() => {
    // ここがコールバック関数
    console.log("Fooがマウントされました！");
    // ↓これがクリーンアップ関数
    return () => {
      console.log("Fooがアンマウントされる！");
    };
  }, []);
  return <p>I am foo</p>;
};
```

このコンポーネント`Foo`は、マウントされたときに`Fooがマウントされました！`と表示し、アンマウントされるときに`Fooがアンマウントされる！`と表示するでしょう。

もう一つ例をみてみましょう。

```tsx
const Foo = () => {
  useEffect(() => {
    // ここがコールバック関数
    console.log("foo!");
    // ↓これがクリーンアップ関数
    return () => {
      console.log("bye!");
    };
  });
  return <p>I am foo</p>;
};
```

今度の例は`useEffect`の第2引数が無いので、レンダリングのたびにコールバック関数が呼ばれます。クリーンアップ関数は、次のコールバック関数が呼ばれる前（または`Foo`がアンマウントされる前）に呼び出されます。`Foo`を最初にマウントしたとき`foo!`と表示され、次に`Foo`が再レンダリングされたときは`bye!`→`foo!`の順に表示されます。最後に`Foo`がアンマウントされるときは`bye!`だけ表示されます。

ここに、さらにDOMの反映や画面の反映の話が関わってきます。レンダリング・コールバック関数・クリーンアップ関数に関してまとめると次のような流れとなります。まず、最初のレンダリング（レンダリング1）が発生したときは、すでに説明したように次の流れとなります。

- レンダリング1 → レンダリング1の結果がDOMに反映 → DOMが画面に反映 → コールバック関数1 

このコールバック関数1がクリーンアップ関数1を返したとしましょう。すると、次のレンダリング（レンダリング2）が発生したときは次の流れとなります。

- レンダリング2 → レンダリング2の結果がDOMに反映 → **クリーンアップ関数1** → DOMが画面に反映 → コールバック関数2

このことから分かることは、クリーンアップ関数1で長い処理を実行することで、レンダリング2の結果の画面への反映を遅らせることができるということです。試しにクリーンアップ関数で先ほどの`run1sec`を使ってみましょう。

```tsx
const Foo = () => {
  useEffect(() => {
    return () => {
      run1sec();
      console.log("bye!");
    };
  });
  return <p>I am foo</p>;
};

function run1sec() {
  const start = performance.now();
  while (performance.now() - start < 1000);
}
```

こうすると、レンダリング2の結果が画面に反映されるのを1秒遅らせることができます。実際、Fooを再レンダリングしたりアンマウントしたりすると、それが画面に反映されるまで1秒かかるでしょう。

なお、クリーンアップ関数が呼ばれるタイミングは、「コンポーネントが再レンダリングされた場合」と「コンポーネントがアンマウントされた場合」で少し異なります。「次のレンダリングが発生した場合」、すなわちレンダリング2の後も`Foo`はマウントされたままである（かつ`Foo`も再レンダリングされる）場合は先ほどの再掲ですが以下のようになります。

- レンダリング2 → レンダリング2の結果がDOMに反映 → **クリーンアップ関数1** → DOMが画面に反映 → コールバック関数2

一方、レンダリング2の結果として`Foo`が画面から消える（アンマウントされる）場合もあります。その場合は次の流れとなります。

- レンダリング2 → **クリーンアップ関数1** → レンダリング2の結果がDOMに反映 → DOMが画面に反映

つまり、アンマウントされる場合はレンダリング2の結果がDOMに反映される前（言い換えればDOMにまだ`Foo`の中身が残っている状態）に実行されるのです。このことに気づかずにReactを使っている方も多いと思いますが、改めて示されると何だか不整合がある感じがしますね。

## React 17におけるクリーンアップ関数のタイミングの変更

実は、React 17ではクリーンアップ関数のタイミングが変更され、同時に上述の不整合も解消されます。これが`useEffect`の破壊的変更です。React 16とReact 17を比較すると次のようになります。

**再レンダリング後もコンポーネントが残っている場合**

- React 16: レンダリング2 → レンダリング2の結果がDOMに反映 → **クリーンアップ関数1** → DOMが画面に反映 → コールバック関数2
- React 17: レンダリング2 → レンダリング2の結果がDOMに反映 → DOMが画面に反映 → **クリーンアップ関数1** → コールバック関数2

**アンマウントされる場合**

- React 16: レンダリング2 → **クリーンアップ関数1** → レンダリング2の結果がDOMに反映 → DOMが画面に反映
- React 17: レンダリング2 → レンダリング2の結果がDOMに反映 → DOMが画面に反映 → **クリーンアップ関数1**

主に、React 17では2つの変更が入ったと見ることができます。第一に、クリーンアップ関数はレンダリング2の結果の新しいDOMが画面に反映されてから呼ばれるようになります。これにより上の例でコールバック関数1が`run1sec`を実行しても、React 17ならばその前に画面への反映が済んでいるので、影響がありません。この変更の利点として、上の例から明らかなように、クリーンアップ関数の実行に時間がかかってもレンダリング結果の画面への反映が遅れないようになります。この違いを確かめられるCodeSandboxを用意しました。React 16をReact 17に変えることで挙動の違いを実感できます。

- https://codesandbox.io/s/react17-useeffect-article-1-xdce8

第二に、React 17ではコンポーネントがアンマウントされる場合も同様に、DOMが画面に反映された後にクリーンアップ関数が呼び出されるように変更されます。この場合は特に、クリーンアップ関数が呼ばれる際のDOMの状態が異なります。従来はレンダリング1がDOMに反映された状態でクリーンアップ関数が呼び出されていましたが、React 17ではレンダリング2がDOMに反映された状態で呼び出されます。

この変更により、アンマウントか否かによってコールバック関数が呼び出される際のDOM状態が異なるという問題が解消され、React 17は常に「次のレンダリング後」（レンダリング2の後）のDOM状態でクリーンアップ関数が呼び出されることになります。

### 破壊的変更の影響を受ける例

この第二の変更が問題となるのは次のような場合です（Reactの公式サイトにも似たような例が挙げられていますが、ここではより具体的な例を紹介します）。

```tsx
const Foo = () => {
  const pRef = useRef();
  useEffect(() => {
    const handler = (e) => {
      console.log("hi!");
    };
    pRef.current.addEventListener("click", handler);
    return () => {
      pRef.current.removeEventListener("click", handler);
    };
  }, []);
  return <p ref={pRef}>I am foo</p>;
};
```

この例では、`useEffect`を用いて`<p>`にclickイベントを設定しています（本当は`<p onClick={...}>`とした方がいいのですが、例なので大目に見てください）。このように、useEffectのコールバック関数の中でイベントハンドラを設定した場合、クリーンアップ関数でそれを消すというのが典型的なパターンです。`useEffect`の第2引数が`[]`なので、このコンポーネントがマウントされたときにイベントハンドラを設定し、アンマウントされるときにイベントハンドラを削除します。

このコンポーネントはReact 16ではうまく動作しますが、React 17では動作しません。その理由は、React 17ではクリーンアップ関数の中では`pRef.current`が`null`となっているからです。React 17ではコンポーネントがアンマウントされるときもレンダリングがDOMに反映されてからクリーンアップ関数が呼び出されるのでした。つまり、このクリーンアップ関数が呼び出された際に、すでに`<p>`は（画面の）DOMツリーから除去されているのです。Reactの`ref={pRef}`という機能を使っている場合、`p`がDOMツリーから除去された時点で`pRef.current`が`null`に設定されます。

回避策は、コールバック関数の時点で`pRef.current`の内容を変数に退避しておくことです。次の例では変数`p`に`pRef.current`を入れてそれをクリーンアップ関数でも使っています。

```tsx
const Foo = () => {
  const pRef = useRef();
  useEffect(() => {
    const handler = (e) => {
      console.log("hi!");
    };
    const p = pRef.current;
    p.addEventListener("click", handler);
    return () => {
      p.removeEventListener("click", handler);
    };
  }, []);
  return <p ref={pRef}>I am foo</p>;
};
```

クリーンアップ関数の時点で`<p>`がDOMツリーから除去されている点は変わりませんが、`pRef.current`に頼らず`p`という変数に`<p>`を表すDOMオブジェクトを入れておくことによって、DOMツリーから除去された後のp要素のオブジェクトにアクセスすることが可能で、無事に`p.removeEventListener`を呼び出すことができます。尤も、この例の場合DOMから除去された`p`はもう使われないので`p.removeEventListener`を呼び出さずに放置してもいいのですが、いつそれをして良くていつそれをしてはならないのかを理解していない人が真似してしまうと困るので、どんな場合もきちんと後処理をするのも悪い選択肢ではありません。

### クリーンアップ関数の実行順序について

Reactの公式サイトをよく読むと、React 17におけるクリーンアップ関数の実行順序について次のようなことが書いてあります。

> Additionally, React 17 executes the cleanup functions in the same order as the effects, according to their position in the tree. Previously, this order was occasionally different.
>
> （筆者による翻訳）さらに、React17はクリーンアップ関数を、コンポーネントツリーの中での位置に従って、エフェクト（訳注：`useEffect`のコールバック関数）と同じ順序で実行します。以前は、時折順序が異なる場合がありました。

これに関して筆者はいくつか実験してみたのですが、どうも書いてある通りの挙動にならないのでReactのGitHubリポジトリにissueを建ててみました。返事を待っているところです。

- [Bug(17.0.0-rc.1): useEffect cleanup functions not running in the same order as effect functions](https://github.com/facebook/react/issues/19866)

ただ、関連してReact 16と17の間で変化した点を発見したので紹介しておきます。
次のような並びでコンポーネントが並んでいて、それぞれが中で`useEffect`を使っているとしましょう。

```tsx
<Foo1 />
<Foo2 />
<Foo3 />
```

1回の再レンダリングでこれらのコンポーネントが全て再レンダリングされる（`useEffect`のコールバック関数が呼ばれる）場合、React 16での実行順序は以下の順となります。

- `Foo1`のクリーンアップ関数1 → `Foo1`のコールバック関数2 → `Foo2`のクリーンアップ関数1 → `Foo2`のコールバック関数2 → `Foo3`のクリーンアップ関数1 → `Foo3`のコールバック関数2

一方、React 17では次の順序となります。

- `Foo1`のクリーンアップ関数1 → `Foo2`のクリーンアップ関数1 → `Foo3`のクリーンアップ関数1 → `Foo1`のコールバック関数2 → `Foo2`のコールバック関数2 → `Foo3`のコールバック関数2

つまり、React 16では各コンポーネントに対して個別にクリーンアップ→次のコールバックという流れを実行していたのに対して、React 17ではまず全てのクリーンアップ関数を呼び出したあとに次のコールバックを全部呼び出すという流れとなっています。

## まとめ

React 17で予定されている破壊的変更のひとつとして、`useEffect`のコールバック関数・クリーンアップ関数に対する変更を解説しました。

Reactの公式サイトでも言われている通り、破壊的変更といっても実際にアプリケーションに影響が出るケースは多くないでしょう。最後に見たコンポーネント間のエフェクトの順序に関する変更など、よほど黒魔術的なコードを書かないと影響を受けません。

そうはいっても、こういった機序を理解していなければ`useEffect`を用いたコードの正確な理解は困難ですから、ぜひ覚えておきましょう。

また、現在React 16を使用している方は、React 17の破壊的変更の影響を受けてしまうようなコードを書かないように気をつけましょう。
