---
title: "自己補正するコンポーネント: レンダリング中に状態更新する公式テクニックの解釈"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

Reactにおいてプロダクトの品質を高く保つには、Reactのやり方に合ったコードを書くことが重要です。公式ドキュメントの名物ページ「[そのエフェクトは不要かも](https://ja.react.dev/learn/you-might-not-need-an-effect)」には、useEffectの望ましくない使い方と、それに代わるテクニックが紹介されています。

この記事で取り上げるのは、その中でも「[props が変更されたときに一部の state を調整する](https://ja.react.dev/learn/you-might-not-need-an-effect#adjusting-some-state-when-a-prop-changes)」のセクションで紹介されている、レンダリング中にステートを更新するテクニックです。

このテクニックは一見すると奇妙で、知ってはいるけど使っていいのかよく分からないという方も多そうです。しかし、一見突飛に見える記述でも、深く理解すれば実はReactのデザインに完璧に則っているというのがReactの公式ドキュメントの特徴です。技術に対する向き合い方として、公式の説明であっても妄信せず批判的に見ることはとても重要です。しかし、Reactの場合は最終的には「すごく考えたけどやっぱり公式ドキュメントが正しかった」に戻ってきてしまうことがとても多いのです。

この記事では、皆さんがこの境地にたどり着く助けとなるべく、レンダリング中に状態を更新するテクニックについて考えていきます。

## テクニックの概要

まずは、公式ドキュメントで紹介されているテクニックを確認しましょう。以下の例は公式ドキュメントからの引用です。

```jsx
function List({ items }) {
  const [isReverse, setIsReverse] = useState(false);
  const [selection, setSelection] = useState(null);

  // 🔴 Avoid: Adjusting state on prop change in an Effect
  useEffect(() => {
    setSelection(null);
  }, [items]);
  // ...
}
```

これは、選択肢`items`の中からひとつ選択できる機能を提供するコンポーネントの一部です。ありがちなのが、`items`が変わったときに未選択状態に戻すという仕様になっている場合です。このコードでは、`items`が変わるたびに`useEffect`の中で`setSelection(null)`を呼び出して、選択状態をリセットしています。これは良くありません（理由は後述）。

代わりに、公式ドキュメントでは、以下のコードのほうが良いとしています。

```jsx
function List({ items }) {
  const [isReverse, setIsReverse] = useState(false);
  const [selection, setSelection] = useState(null);

  // Better: Adjust the state while rendering
  const [prevItems, setPrevItems] = useState(items);
  if (items !== prevItems) {
    setPrevItems(items);
    setSelection(null);
  }
  // ...
}
```

このコードでは、`prevItems`というステートを用意して、前回の`items`の値を保存しています。そして、レンダリング中に`items`と`prevItems`を比較し、異なっていれば`setSelection(null)`を呼び出して選択状態をリセットしています。これにより、`items`が変わったときに選択状態をリセットするという仕様を満たしつつ、`useEffect`を使わずに済んでいます。

ポイントは、ステートの更新（`setSelection`などの呼び出し）が**レンダリング中**に行われていることです。レンダリング中とは、イベントハンドラや`useEffect`の中ではなく、コンポーネント関数の本体部分（関数が返り値を返すまでの間に通る処理）で行われていることを指します。

:::message
このテクニックは、公式ドキュメントでuseEffectを使うよりはマシなものの、濫用するのではなく他の選択肢を検討すべきとされている点にはご注意ください。例えば、[keyを使ってコンポーネントを再作成する](https://zenn.dev/uhyo/articles/react-key-techniques)方法や、そもそも仕様を見直して状態をリセットする必要がないようにするなどです。
:::

## なぜuseEffectでは良くないのか

この場合になぜuseEffectを使うのが良くないのかは、主に2つの観点で説明できます。

一つは、useEffectの本来の使い方ではないからです。useEffectは「コンポーネントが存在することによる影響」を実装するために使うものであり、「propsなどの変化を検知する」ために使うものではありません。言い換えれば、「`items`が変わったことを検知したい」という動機でuseEffectを使ったとしたら、それはReactの宣言的な考え方を逸脱しており望ましくありません。

もう一つは、単純にパフォーマンスやユーザー体験の観点から良くないからです。useEffectはレンダリングが完了した後に実行されるため、`items`が変わったときに、一瞬だけ「`items`は新しくなったけど`selection`は更新されていない状態」が発生し、DOM更新まで行われてしまいます。これにより、ユーザーが一瞬だけ不整合な状態を目にする可能性もあります（これはuseLayoutEffectを使えば抑制できますが、パフォーマンス的に良くないことは変わりません）。

逆に言えば、レンダリング中に状態を更新するテクニックのほうが、パフォーマンス的にもマシなのです。これについて詳しく見ていきましょう。

## 2種類の方法を比較する

以下のように、3段階にネストしたコンポーネントを考えます。`console.log`で関数コンポーネントが呼び出されたことを確認できるようにしています。

```jsx
const Wrapper: React.FC = () => {
  console.log("Wrapper");
  return (
    <div>
      <List />
    </div>
  );
};

const List: React.FC = () => {
  console.log("List");
  const [value, setValue] = useState(0);

  useEffect(() => {
    console.log("useEffect value:", value);
  });

  return (
    <ul>
      <Item />
      <Item />
      <Item />
    </ul>
  );
};

const Item: React.FC = () => {
  console.log("Item");
  return <li>Item</li>;
};
```

これに対して`<Wrapper />`をレンダリングすると、コンソールには以下のように表示されます（StrictModeの場合関数が2回ずつ呼ばれますが、今回は省略します）。

```
Wrapper
List
Item
Item
Item
useEffect value: 0
```

ここで、`List`コンポーネントに対して、`useEffect`を使って状態を更新する方法と、レンダリング中に状態を更新する方法の2つを実装してみます。

### useEffectを使う方法

```jsx
const List: React.FC = () => {
  const [value, setValue] = useState(0);
  console.log("List", value);

  useEffect(() => {
    if (value === 0) {
      // valueが0なのは嫌なので1にする
      setValue(1);
    }
  });

  return (
    <ul>
      <Item />
      <Item />
      <Item />
    </ul>
  );
};
```

この状態で`<Wrapper />`をレンダリングすると、コンソールには以下のように表示されます。

```
Wrapper
List 0
Item
Item
Item
useEffect value: 0
List 1
Item
Item
Item
useEffect value: 1
```

これは、まずvalueが0の状態でレンダリングが完了し、useEffectが発火したことを示します。その後、setValue(1)が呼び出されて再レンダリングが発生し、valueが1の状態で再度レンダリングが行われています。

### レンダリング中に状態を更新する方法

次は、useEffectを使わずに、レンダリング中に状態を更新する方法です。

```jsx
const List: React.FC = () => {
  const [value, setValue] = useState(0);
  console.log("List", value);

  if (value === 0) {
    // valueが0なのは嫌なので1にする
    setValue(1);
  }

  useEffect(() => {
    console.log("useEffect value:", value);
  });

  return (
    <ul>
      <Item />
      <Item />
      <Item />
    </ul>
  );
};
```

この状態で`<Wrapper />`をレンダリングすると、コンソールには以下のように表示されます。

```
Wrapper
List 0
List 1
Item
Item
Item
useEffect value: 1
```

この結果からは、以下のことが分かります。

まず、Listのレンダリング中にステートを更新したことにより、**Listのレンダリングが中断され、やり直された**ことが分かります。特に、useEffectの場合とは異なり、valueが0のときはListの子のItemコンポーネントのレンダリングが行われていません。即座に新しいステートでListのレンダリングがやり直されて、valueが1の状態でItemコンポーネントのレンダリングが行われています。

さらに、useEffectの発火は一度だけであり、valueが1の状態でのみ発火していることも分かります。valueが0の状態でレンダリングが完了することはなく、したがってuseEffectも発火していません。

このように、2つの方法ではレンダリングの挙動が大きく異なっていることが分かります。レンダリングの最中にステートを更新した場合、レンダリングを完遂することなくやり直しがかかります。valueが0の状態のレンダリングは中断されたため、当然その結果がDOMに反映されることもなく、それに対してuseEffectも発火しません。

これが公式ドキュメントで説明されているパフォーマンスの違いです。

### 補足: 自身のステートしか更新できない

このテクニックには制約があります。レンダリング中に状態を更新できるのは、そのコンポーネント自身のステートに限られます。親コンポーネントや兄弟コンポーネントのステートを更新することはできません。例えば、以下のようなコードは動作しません。

```jsx
const Parent: React.FC = () => {
  const [value, setValue] = useState(0);
  return <Child setValue={setValue} />;
};

const Child: React.FC<{ setValue: React.Dispatch<React.SetStateAction<number>> }> = ({ setValue }) => {
  // ❌ これは動作しない
  setValue(1);
  return <div>Child</div>;
};
```

実際に`<Parent />`をレンダリングすると`setValue(1);`のところで以下のエラーが発生します。

> Cannot update a component (`Parent`) while rendering a different component (`Child`). To locate the bad setState() call inside `Child`, follow the stack trace as described in https://react.dev/link/setstate-in-render

このことから、レンダリング中に状態を更新する機能は、自身のステートに対してのみ使用できることが分かります。

## Reactのデザインとの整合性

この機能を、筆者は「**コンポーネントの自己補正**」ができる仕組みとして理解しています。コンポーネントは、自身の状態が不整合な状態にあると判断した場合に、自動的にその状態を修正することができるのです（公式ドキュメントでは「stateの調整」と表現されています）。

それを実現する仕組みとして、**レンダリングのやり直し**をするようになっています。結果的に、Listを1回レンダリングするために関数としてListが2回呼び出されることになっています。

このように、レンダリングが完了するまでに複数回関数コンポーネントが呼び出されることは、最近のReactでは普通のことです。そもそも、Suspenseもそのような仕組みで実現されていますね。コンポーネントのレンダリング中（関数実行中）にサスペンドが発生したらそのコンポーネントのレンダリングは中断されます。データが取得できたらレンダリングがやり直されるのです。

そもそも、StrictModeでは開発環境において関数コンポーネントが2回呼び出されるようになっており、この挙動はReact 16の時点で存在していました。React 16あたりでは「1回関数が実行される＝1回レンダリングされる」が基本の挙動でしたが、その時点から本来はそうとは限らないという考え方が確立されていたのです。

UIライブラリとしてのReact（特に18以降）の際立った特徴は、**高度なレンダリングのスケジューリング機能**にあります。つまり、最終的なUXを最適化するために、Reactは「いつどこでどの関数コンポーネントを呼び出すか」のようなことを制御しています。言い換えると、我々は「コンポーネントが関数としていつ呼び出されるか」の制御をReactに委ねています。だからこそ、関数コンポーネントが1レンダリング中に複数回呼び出されることにも備える必要があります。

以上のように考えると、レンダリング中にステートを自己補正してレンダリングをやり直させても、Reactの考え方からそんなに逸脱していないことが分かります。

Reactにおいて、関数コンポーネントの呼び出しが1回増える程度のことでパフォーマンスに与える影響は微々たるものです。それを前提に、Reactは宣言的にUIを記述できる仕組みとUXの最適化を両立させているのです。また、前述のようにこの機能は自分自身のステートを更新することしかできません。これもパフォーマンスへの悪影響を最小限に留めるための制約です。

## コンポーネントの自己補正を意識して見直してみる

冒頭で公式ドキュメントから引用したコードをもう一度見てみましょう。「コンポーネントの自己補正」という考え方を踏まえてこのコードを解釈します。

```jsx
function List({ items }) {
  const [isReverse, setIsReverse] = useState(false);
  const [selection, setSelection] = useState(null);

  // Better: Adjust the state while rendering
  const [prevItems, setPrevItems] = useState(items);
  if (items !== prevItems) {
    setPrevItems(items);
    setSelection(null);
  }
  // ...
}
```

このテクニックで使われている`prevItems`というステートは、名前のとおり「前回のitems」を保存するために使われています。

しかし、自己補正という考え方をすると、別の見方ができます。それは、`prevItems`は「**どのitemsに対してリストをレンダリングしているとListが認識しているか**」を表しているということです。

通常、`items`と`prevItems`は一致しています。しかし、親コンポーネントが新しい`items`を渡してきた場合、`items`と`prevItems`は不一致になります。これが**不整合**です。これは、「Listにとっては選択肢一覧は`prevItems`のつもりなのに、親からの指示は異なっている」という状態になります。

Listコンポーネントはこの不整合を検知して自己補正を行います。その過程で、「`selection`はあくまで`prevItems`に対する選択状態であり、新しい`items`に対しては無効である」という認識に基づき、`selection`をリセットします。そして、`prevItems`を新しい`items`で更新します。これにより、Listコンポーネントは自身の状態を親からの指示（新しい`items`）に合わせて自己補正したことになります。

この考え方に則ると、`prevItems`というステート名は若干違うように思えます。むしろ`currentItems`のような名前のほうが、Listコンポーネントが「現在レンダリングしているitems」を認識していることをより正確に表しているように思えます。直すとすれば以下のようになります。

```jsx
function List({ items }) {
  const [isReverse, setIsReverse] = useState(false);
  const [selection, setSelection] = useState(null);
  const [currentItems, setCurrentItems] = useState(items);

  if (items !== currentItems) {
    setCurrentItems(items);
    setSelection(null);
  }

  return (
    <ul>
      {currentItems.map((item) => (
        // ...
      ))}
    </ul>
  );
}
```

`prevItems`という名前だと「状態の変化を検知する」というある種手続き的なニュアンスが強くなりますが、`currentItems`という名前にするとより宣言的で、自己補正という考え方に合った命名になりますね。それにも関わらず公式ドキュメントで`prevItems`という名前が使われているのは、「状態の変化を検知する」というメンタルモデルのほうが多くのReact利用者にとっては理解しやすいからかもしれません。

しかし、この記事で**自己補正**という考え方を知った皆さんは、ぜひ`currentItems`のような命名にも挑戦してみてください。コンポーネントから手続き的なメンタルモデルを排除することは、コードの理解を容易にし、保守性を高める助けになります。

ただし、あまりむやみに連発するテクニックではないことは改めて強調しておきます。そもそも、`currentItems`なんて不要で親から渡された`items`を唯一の真実として扱えるならばそれに越したことはありません。今回は、コンポーネントの仕様上「`selection`というステートは現在のitemsに依存する」（言い換えれば、itemsが変化したら`selection`との不整合を検知しなければならない）という設計が必要であり、その結果として親から渡された`items`とは別に自身の中に真実の認識（`currentItems`）を持つ必要があるため、このテクニックが必要になっているのです。

### 補足: 純粋性は大丈夫なのか

ご存じのとおり、Reactでは関数コンポーネントは“純粋”であることが求められます。つまり、同じpropsとstateに対しては常に結果を返すべきであり、副作用を持つべきではないということです。

そうなると、「レンダリング中に状態を更新するのは副作用ではないのか？」という疑問が湧くのではないでしょうか。

筆者の解釈では、これは問題ありません。なぜなら、「状態を更新して、レンダリングのやり直しという指示を出した」ということも、レンダリングの**結果の一種**だからです。Reactにおける“純粋”の概念は一般的な関数型プログラミングにおける“純粋”と比べると拡大された概念であり、関数の返り値以外にも色々な「結果」があります。関数がサスペンドする（内部的な挙動としては例外を投げている）ことも「結果」のひとつです。

この場合、純粋性に反するのは、「同じpropsとstateなのに、レンダリングをやり直ししたりしなかったりすること」です。一方、この記事で紹介したようなコードでは、同じpropsとstateに対しては常に同じ挙動（レンダリングのやり直しをするかしないか）が保証されています。したがって、Reactが求める純粋性は保たれていると言えます。

上述のようにステートの更新が自分自身のステートにしか許されていないという点も、この考え方を補強します。他のコンポーネントのステートを更新できてしまうと、コンポーネントのレンダリング結果がそのコンポーネントに完結しなくなり、さすがに副作用になってしまうからです。

## 注意点: 返り値は返す必要がある

最後に、このテクニックの注意点を紹介します。それは、レンダリングのやり直しを指示したとしても、**コンポーネント関数は必ず返り値を返す必要がある**ということです。例えば以下のようなコードはうまくいきません。

```jsx
const List: React.FC = () => {
  const [value, setValue] = useState(0);
  console.log("List", value);

  if (value === 0) {
    // valueが0なのは嫌なので1にする
    setValue(1);

    throw new Error("ぎゃ～～～！");
  }

  useEffect(() => {
    console.log("useEffect value:", value);
  });

  if (value === 0) {
    return null;
  }

  return (
    <ul>
      <Item />
      <Item />
      <Item />
    </ul>
  );
};
```

レンダリングをやり直す場合、コンポーネントの返り値は使われないので意味が無いような気もします。そのため、レンダリングをやり直す場合はその場でエラーを投げて中断してもいいのではないかとも思えます。

しかし、上記のようにした場合、「ステートを更新したこと」よりも「エラーが投げられたこと」が優先されてしまいます。つまり、レンダリングの結果は「ステート更新してやり直し」ではなく「エラーが発生」になってしまいます（エラーがエラーバウンダリーに補足されます）。したがって、レンダリングのやり直しを指示した場合でも、返り値をちゃんと返す必要があります。

また、次のようにするのもだめです。

```jsx
const List: React.FC = () => {
  const [value, setValue] = useState(0);
  console.log("List", value);

  if (value === 0) {
    // valueが0なのは嫌なので1にする
    setValue(1);
    return null; // ❌
  }

  useEffect(() => {
    console.log("useEffect value:", value);
  });
  return (
    <ul>
      <Item />
      <Item />
      <Item />
    </ul>
  );
};
```

これはエラーを投げるのではなく、レンダリングのやり直しが必要と判明した時点でnullを返していますが、こうすると**フックのルールに反してしまう**のでだめです。

フックのルールでは、条件に応じてフックが呼び出されたり呼び出されなかったりしてはいけません。上記のコードでは、`value`が0のときは`useEffect`が呼び出されず、1のときは呼び出されるため、フックのルールに反しています。レンダリングのやり直しが必要な場合でもこのルールを守る必要があります。

以上のことから、レンダリングのやり直しを指示した場合でも、コンポーネント関数は必ず返り値を返し、かつフックのルールを守る必要があることが分かります。もし、不整合が発生しており返り値に意味が無い場合は、次のようにするのが良いでしょう。

```jsx
const List: React.FC = () => {
  const [value, setValue] = useState(0);
  console.log("List", value);

  if (value === 0) {
    // valueが0なのは嫌なので1にする
    setValue(1);
  }

  useEffect(() => {
    console.log("useEffect value:", value);
  });

  if (value === 0) {
    // 不整合な状態なので何も表示しない
    return null;
  }
  return (
    <ul>
      <Item />
      <Item />
      <Item />
    </ul>
  );
};
```

このように、フックを全部過ぎてからnullを返すのは大丈夫です。レンダリングのやり直しを指示した時点でListの返り値のJSXは使われませんので、nullを返しても問題ありません。

このやり方だと`value === 0`のような条件分岐を2箇所に書く必要がある（あるいは、それを避けるならフラグみたいなものを用意する必要がある）のでやや冗長ですが、Reactの制約なのでしょう。個人的には外そうと思えばこの制約は外せる気がしていますが。

## まとめ

この記事では、レンダリング中に状態を更新するテクニックについて、実はReactのデザインに反することなく利用可能なテクニックであることを紹介しました。

**コンポーネントの自己補正**という考え方をすることで、Reactの設計との整合性を理解しやすくなります。コンポーネントは、自身の状態が不整合であると判断した場合に、自動的にその状態を修正することができるのです。