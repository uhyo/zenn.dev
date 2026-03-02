---
title: "Async React時代の宣言的UI: デバウンスの例"
emoji: "🏀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

**宣言的UI**とは何か、皆さんは答えられるでしょうか。

「あーあの、DOM更新を直接プログラムに書くんじゃなくて、JSXとかであるべき状態を宣言したらライブラリが自動的に差分適用とかでDOMを更新してくれるやつでしょ？」

もちろん、このような答えは間違いではありません。しかし、特に**Async React**の時代においては、Reactの考えはさらに先を行っているようです。

究極的には、宣言的UIは、**やりたいことをロジックとして記述するだけで、具体的なことや細かい最適化はよしなにやってくれる**ものだと筆者は考えています。上述のようなDOMの更新の話はその一例にすぎません。

- **やりたいこと:** 特定の形のDOMを画面に表示したい。
- **具体的なこと:** Reactランタイムがコンポーネントツリーを実行してDOMを更新する。
- **最適化:** DOMをいい感じに差分更新する。

また、`useState`などといったステート管理の概念を持ち出しているのも、宣言的UIだと捉えられます。

- **やりたいこと:** アプリケーションのロジックを、DOM操作ではなく単なるステートの計算として記述したい。
- **具体的なこと:** Reactランタイムがステートの変更に応じてコンポーネントを再レンダリングする。
- **最適化:** ステート更新のバッチ処理など。

このように、実は宣言的UIは、UIの実装における幅広い場所に適用できる概念です。そして、最近のAsync Reactの動きからすると、Reactはこの宣言的UIの考えをさらに推し進めているように見えます。ちなみに、Async Reactについては筆者の以下の発表でも紹介しています。

https://speakerdeck.com/uhyo/react-19shi-dai-nokonponentoshe-ji-besutopurakuteisu

もしあなたがReactを使っていて、「具体的なこと」や「最適化」を自分で書かなければいけなかったしましょう。その場合、極論、その理由は以下のどちらかです。

- あなたが宣言的UIを十分に実践できていない。
- React本体の機能やエコシステムがまだ未熟で、宣言的UIを実践するための土台が不足している。

Async Reactワーキンググループの目標はAsync Reactの教育と普及とされていますが、「教育」が1つ目の理由をカバーし、「普及」が（エコシステムのライブラリ等の対応を通じて）2つ目の理由をカバーすることになるでしょう。

理想的な宣言的UIの世界では、Reactを使うのはすごく簡単なはずです。小手先のあれこれを自分でやらなくてもReactがよしなにやってくれて、実装者は本質的なロジックの記述に集中できるからです。

Reactが「難しい」と感じる人には、宣言的UIそのものを難しいと感じる人と、Reactを実用レベルで使うための細かな実装テクニックが難しいと感じる人の両方がいるのではないかと思います。宣言的UIの適用範囲の広がりに応じて、後者の難しさは減っていくはずです。

この記事では、自分でやらなくてもReactがよしなにやってくれることの例として、**デバウンス**を取り上げてみたいと思います。

正確には、デバウンスそのものをReactがやってくれるというよりは、それを使って解決したい問題に対して、React本体がどのように対応しているのかという話です。

実のところ、以下の話は[useDeferredValue](https://ja.react.dev/reference/react/useDeferredValue)の基本的なユースケースの話（+少し応用）です。そのため、useDeferredValueをすでに使いこなせる人にとっては得るものがあまりないかもしれません。

それでも、この話をAsync Reactや宣言的UIという文脈で語ることには意味があると思い、この記事を書きました。つまり、「これも宣言的UIなんだよ」ということです。

## 今回の題材

今回の題材は、1万個のUUIDの一覧から部分一致の文字列検索ができるというものにしましょう。とても実践的な例ですね。

ざっくりこのような実装です。

```tsx
const App: React.FC = () => {
  const [filter, setFilter] = useState("");
  const filtered = filter ? uuids.filter((id) => id.includes(filter)) : uuids;

  return (
    <>
      <h2>10,000 UUIDs</h2>
      <input
        type="text"
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
        placeholder="Filter UUIDs..."
      />
      <p>{filtered.length} results</p>
      <FilteredList filteredUuids={filtered} filter={filter} />
    </>
  );
};
```

:::details FilteredListの実装

```tsx
interface Props {
  filteredUuids: readonly string[];
  filter: string;
}

export const FilteredList: React.FC<Props> = ({ filteredUuids, filter }) => {
  return (
    <ul>
      {filteredUuids.map((id) => (
        <li key={id}>{[...highlight(id, filter)]}</li>
      ))}
    </ul>
  );
};

function* highlight(text: string, filter: string) {
  if (!filter) {
    yield text;
    return;
  }

  const parts = text.split(filter);
  for (let i = 0; i < parts.length; i++) {
    yield parts[i];
    if (i < parts.length - 1) {
      yield <mark key={i}>{filter}</mark>;
    }
  }
}
```

:::

余談ですが、「`filtered`の計算は`useMemo`を使ったほうがいいんじゃないの？」と思った方がいるかもしれません。しかし、今回は**React Compiler**を使っていることを前提にしているため`useMemo`は使用しません。これも、React Compilerによってより理想の宣言的UIに近づいた例です。自ら`useMemo`を書くという、本来やりたいことというよりは最適化の部類のコードを自分で書く必要がなくなっていますね。

さて、この実装を実際に動かしてみると、結構重いです。入力欄に1文字（`a`とか）を入力すると、それが反映される（inputに実際に`a`が反映される）まで0.5秒とかかかります。

重い理由は、`filter`が変わったことによって`filtered`が変わり、`FilteredList`の新しいリストの再レンダリングをしないといけないからです。特に、最初の1文字を入力した場合、フィルタリング後は8500件くらいのUUIDが残るため、`FilteredList`は8500件のリストアイテムを再レンダリングすることになります。

## 昔ながらの対処法: デバウンス

**デバウンス**とは、イベントの発生から一定時間が経過するまで処理を遅延させるテクニックのことです。これを使うと、ユーザーが入力を完了するまでフィルタリング処理を遅らせることができます。

```tsx
const Debounced: React.FC = () => {
  const [filter, setFilter] = useState("");
  const [debouncedFilter, setDebouncedFilter] = useState(filter);

  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setFilter(value);
    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }
    timerRef.current = setTimeout(() => {
      setDebouncedFilter(value);
    }, 300);
  };

  const filtered = debouncedFilter
    ? uuids.filter((id) => id.includes(debouncedFilter))
    : uuids;

  return (
    <>
      <h2>10,000 UUIDs</h2>
      <input
        type="text"
        value={filter}
        onChange={handleChange}
        placeholder="Filter UUIDs..."
      />
      <p>{filtered.length} results</p>
      <FilteredList filteredUuids={filtered} filter={debouncedFilter} />
    </>
  );
};
```

この実装では、`filter`と`debouncedFilter`の2つのステートを持っています。ユーザーが入力するたびに`filter`はすぐに更新されますが、`debouncedFilter`は300msの遅延の後に更新されます。これにより、ユーザーが入力した瞬間は`debouncedFilter`はまだ古い値のままなので、再レンダリングに時間がかかりません。そのため、ユーザーの入力がスムーズになります。

問題点は、こういう実装は**宣言的UIっぽくない**ということです。われわれの本当に**やりたいこと**は、「ユーザーの入力に応じてフィルタリングされたリストを表示すること」です。デバウンスとかは**最適化**の話であり、こういう最適化はReactが勝手にやってくれるのが理想的です。

## useDeferredValueによる対処

そこで、`useDeferredValue`の出番です。これを使うと、より理想の宣言的UIに近い実装ができます。

```tsx
const Deferred: React.FC = () => {
  const [filter, setFilter] = useState("");
  const deferredFilter = useDeferredValue(filter);
  const filtered = deferredFilter
    ? uuids.filter((id) => id.includes(deferredFilter))
    : uuids;

  return (
    <>
      <h2>10,000 UUIDs</h2>
      <input
        type="text"
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
        placeholder="Filter UUIDs..."
      />
      <p>{filtered.length} results</p>
      <FilteredList filteredUuids={filtered} filter={deferredFilter} />
    </>
  );
};
```

最初の実装に比べると、`const deferredFilter = useDeferredValue(filter);`の行を追加して、フィルタリング結果の表示には`deferredFilter`を使うようにしただけです。

こうすることで、デバウンスの例に近いパフォーマンスを実現します。この場合、ユーザーが入力した結果は即座に`filter`に反映されますが、`deferredFilter`は**同時には更新されません**。よって、すみやかに再レンダリングが完了し、ユーザーの入力がスムーズになります。

その後`filter`の変化に合わせて`deferredFilter`の更新がかかります。ポイントがもう1つあり、この`deferredFilter`の更新は**トランジション**（優先度の低い更新）として扱われるということです。

つまり、`deferredFilter`の再レンダリング（＝フィルタリング結果の再レンダリング）は相変わらず時間のかかる処理ですが、もしその処理の最中にユーザーが再び入力した場合（`filter`がさらに更新された場合）、トランジションは中断されて、`filter`を画面に反映する処理が優先されます。これにより、2文字目以降の入力もスムーズになるはずです（なるはず、というのは、後述しますがこの実装だとまだ完璧ではないためです）。

### 宣言的UIとしてのuseDeferredValueの解釈

`useDeferredValue`により、デバウンスの場合と同様の処理（しかも、300ミリ秒とかいう固定値に頼る必要もない！）をよりシンプルに行うことができるようになりました。

しかし、読者の中にはこのように思うかもしれません。

「確かにシンプルにはなったけど、最適化の道具をReactが用意してくれただけであって、宣言的UIの理想とやらに近づいたわけではないのでは？」

**いい質問ですね！** ✨😁👉　実は、`useDeferredValue`は挙動を見れば確かに最適化のツールに見えますが、より宣言的な「**意味**」を見出すことができます。それは**一貫性を敢えて崩すこと**です。

ライブラリとしてのReactが提供してくれる機能として、**一貫性**があります。それは**中途半端なUIをユーザーに見せないこと**です。例えば、「ステート更新してDOMを更新している途中、画面の上半分だけ更新済みで下半分が未更新の状態がユーザーに見えちゃった」みたいなことを起こさないということです。その結果として、1回のレンダリングで発生したコンポーネントツリーは必ずまとめてコミットされることになります。

一番最初の何も工夫していない例では、レンダリング結果に`<input value={filter} />`と`<FilteredList filteredUuids={filtered} />`の両方が含まれています。`filter`の値が更新されたら、一貫性を保つために、両方を完全にレンダリングしきってから画面に反映されることになります。

つまり、`FilteredList`のレンダリングという重い処理を行わないと`<input value={filter} />`の更新もユーザーに見えないことになります。これが、最初の実装でユーザーの入力が重く感じられる理由です。

一貫性はReactの基本的な性質ですから、最初の実装は「**我々エンジニアが、やりたいこと・UIの仕様の一部として、一貫性を保つことをコードを通じて指示した**」と解釈できます。その結果、あのような重い挙動にならざるを得なかったのです。

一方、`useDeferredValue`を使った実装では、`filter`と`deferredFilter`を使い分けることによって、**この2つの一貫性を保たなくていいこと**を明示したのです。これによりUIの仕様が変わり、その中でReactがよしなに最適化をしてくれた結果、ユーザーの入力がスムーズになりました。

このように、`useDeferredValue`の宣言的UIにおける位置づけは、「**どこに一貫性が必要で、どこに必要でないのか**」というUIの仕様を示すためのものだと解釈できます。

## おまけ: さらなる最適化

上記の`useDeferredValue`による実装でも、素早く入力しているとまだ引っかかりを感じることがあります。その意味で、まだ**最適化**の余地がある実装です。

トランジションによる最適化をしている場合、UXを良くするコツは**コンポーネントを細かく分けること**にあります。今回の場合、上記の`FilteredList`が分けられておらず、1コンポーネントで多くの仕事をしすぎているのが問題です。

というのも、Reactはタスクを作業単位 (unit of work) に分割して処理することで、トランジションの中断を可能にしています。分割の最小単位はコンポーネントであるため、1コンポーネントで作業をしすぎると中断ができず、トランジションの効果が薄れてしまいます。

例えば、`FilteredList`をこのようにすると入力のスムーズさがさらに改善します。

```tsx
export const OptimizedFilteredList: React.FC<Props> = ({
  filteredUuids,
  filter,
}) => {
  return (
    <ul>
      {filteredUuids.map((id) => (
        <li key={id}>
          <Highlighted text={id} filter={filter} />
        </li>
      ))}
    </ul>
  );
};

const Highlighted: React.FC<{ text: string; filter: string }> = ({
  text,
  filter,
}) => {
  return <>{[...highlight(text, filter)]}</>;
};
```

`highlight`関数を呼び出すという仕事が、`FilteredList`の中で全部やるのではなく`Highlighted`コンポーネントに切り出しました。これにより、8500個のハイライト作業の途中でもトランジションを中断できるようになり、反応性がさらに向上します。

さらに改善するならば、`FilteredList`の中で数千個の`Highlighted`子コンポーネントのJSX Elementを全部一気に作るのは重いので、このようにチャンクごとのコンポーネントに分割するのもいいでしょう。

```tsx
export const OptimizedFilteredList: React.FC<Props> = ({
  filteredUuids,
  filter,
}) => {
  const chunkSize = 100;
  const chunks = [];
  for (let i = 0; i < filteredUuids.length; i += chunkSize) {
    chunks.push(filteredUuids.slice(i, i + chunkSize));
  }
  return (
    <ul>
      {chunks.map((chunk, index) => (
        <Chunk key={index} uuids={chunk} filter={filter} />
      ))}
    </ul>
  );
};

const Chunk: React.FC<{ uuids: readonly string[]; filter: string }> = ({
  uuids,
  filter,
}) => {
  return (
    <>
      {uuids.map((id) => (
        <li key={id}>
          <Highlighted text={id} filter={filter} />
        </li>
      ))}
    </>
  );
};
```

ここまでやれば、入力で引っかかる感じはほとんどなくなるはずです。

でも、思いますよね。

「いや結局最適化してない？　これも宣言的UIというのは無理あるよね？」

👈**それ、するどい指摘。** 全くもってその通りです。

この話は、Reactもまだ完璧ではないということを示すために説明しました。本当は、コンポーネント単位といわずもっと自由に仕事を分割できればいいのです。もしかしたら、React Compilerがものすごく進化したらできるかもしれないし、できないかもしれません。

この側面ではReactは宣言的UIの理想を体現できていないので、このような最適化を行わないといけません。

## まとめ

この記事では、Async Reactの文脈で**宣言的UI**について考察しました。特にReactにおいては、単にライブラリがDOMを更新してくれるというだけではなく、**宣言的UIはもっと広くて大きな概念である**ということです。

この記事で`useDeferredValue`の使い方を説明されても、やはりReactは難しいと思われるかもしれません。しかし、少なくとも、「小手先のテクニックの難しさ」ではなく「**宣言的UIをちゃんとやることの難しさ**」に昇華されていれば、筆者としては幸いです。

実際、この種の難しさは他にもあります。例えば、Suspenseバウンダリをどこにどのように配置すればいいのか、という問題です。

いずれにせよ、Reactは宣言的UIのライブラリです。Reactを使いこなすにせよ、批判するにせよ、宣言的UIの文脈に乗って議論ができればその意義も増すのではないかと思います。

今回は具体例としてデバウンスを取り上げました。実際には、ネットワークアクセスをデバウンスしている場合とか応用形もあるのですが、その話はまたいずれ記事にできればと思います。