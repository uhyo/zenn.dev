---
title: "RecoilとRxJSってどう違うの？　共通点は？　調べてみました！"
emoji: "🆚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "recoil"]
published: true
---

皆さんこんにちは。筆者は最近Recoilを推す記事を量産しています。その成果か、Recoilは非同期処理を交えたロジックを書くのが得意であるということは以前よりも知られるようになりました。その次のステップの話題としてよく見られるのが「**Rxと似ている**」「**Rxとどこが違うの？**」といったものです。Rx (Reactive Extensions)、とくにフロントエンドの文脈では[RxJS](https://rxjs.dev/)ですが、これは非同期処理を交えたロジックを記述できるという点で確かにRecoilと類似しています。

そこで、今回はRecoilとRxJSの共通点や違いについて、具体例も交えつつ解説します。

# コンセプトから見るRecoilとRxJSの共通点・相違点

RxJSの特徴については、[RxJSのイントロダクション](https://rxjs.dev/guide/overview)にわかりやすく書いてあります。

> RxJS is a library for composing asynchronous and event-based programs by using observable sequences. It provides one core type, the [Observable](https://rxjs.dev/guide/observable), satellite types (Observer, Schedulers, Subjects) and operators inspired by `Array` methods (`map`, `filter`, `reduce`, `every`, etc) to allow handling asynchronous events as collections.

ちなみに、ここでcore typeとして述べられているObservableについては、[次のように説明されています](https://rxjs.dev/guide/observable)。

> Observables are lazy Push collections of multiple values.

RxJSの説明を見ると、「非同期・イベントベースのプログラムを、Observableを使って合成的に記述するためのライブラリ」であると書いてあります。このうち、半分はRecoilと一致しており、半分は異なります。

具体的には、「非同期」「合成的 (composingをこう意訳しています)」という点がRecoilと共通しています。Recoilにおいては非同期処理は非同期selectorという形で基本的な要素となっています。また、Recoilではatomやselectorといった小さな構成要素を繋ぎ合わせてロジックの全体を構成します。

一方で、「イベントベース」という概念はRecoilのアイデアとは異なります。また、Observableについても「複数の値のコレクションである」という点が強調されていますが、Recoilのatomやselectorはそのようなものではなく、これらは常に現在の値という1つだけの値を保持します。その結果として、selectorのロジックでは自身の過去の値にアクセスすることはできず、そのようなutilも提供されていません。この点は、operator（Observableに対する変換などのロジックを記述するRxJSの構成要素）自身が状態を持ち得て[buffer](https://rxjs.dev/api/index/function/buffer)や[pairwise](https://rxjs.dev/api/index/function/pairwise)といったoperatorが提供されているRxJSとは異なります。

言い方を変えれば、Recoilにおいて構築されるデータフローグラフ（atomやselectorたちが繋がって構成される状態たちの全体のこと）は、状態を持つのはatomだけであって、それに連なるselectorはいわゆる純粋関数です[^note_async_selector]。一方で、RxJSにおいてはObservableに対する変換や計算を表すoperatorひとつひとつが状態を持つ可能性があります。これはメンタルモデルの大きな違いです。Recoilと同じ役割を担わせると想定すると、RxJSのほうがより複雑なロジックを記述できる一方で、複雑性も高いと評価できます。

[^note_async_selector]: 厳密には非同期処理が関わるとちょっと話が変わってきますが、RecoilはReactのSuspenseを土台に、非同期処理があっても純粋に見えるような仕組みになっています。

# 実用観点で見るRecoilとRxJSの共通点・相違点

以上はアイデアレベルの話でしたが、それはそれとして、筆者も実際のところRecoilの書き味はRxJSと似てるなあと思っています。そもそも、RxJSでも[map](https://rxjs.dev/api/operators/map)のように内部状態を持たないoperatorを使えば上述の違いは気になりません。そのため、アプリケーションロジックを記述するという目的に着目すれば似たような書き心地になることが期待されます。

Recoilのselectorは上流から流れてきた値を変換して下流へ流すようなものとして捉えることができますから、そう考えればRxJSのoperatorと似ていますね。また、Recoilにおいてはそのように作られたステートというのは最終的にReactコンポーネントに渡されます。ReactコンポーネントはRecoilのステートに対してsubscribeし、変化があるたびに再レンダリングされます。よりRxJSに寄ったの視点からは、Recoilのデータフローグラフはステートの更新を通知するイベントのストリームだと見なせなくもありません。selectorなどを経由して最終的にイベントがReactコンポーネントにたどり着けば、その値でReactコンポーネントが再レンダリングされることになります。

このように考えるとやっぱりRecoilとRxJS同じじゃんと思いそうですが、ひとつ決定的な違いがあります。それは、Recoilが持つ**ステートの一貫性**という性質です。これは、Recoilの外側から観察されるデータフローグラフの状態は全体が一貫してある特定の時点の状態であり、古い状態と新しい状態が入り混じったものが外側から観測されることはないという意味です。

具体的な話としては、非同期処理の扱いが大きく異なります。RxJSでは非同期的な処理を変換を行う場合、元のObservableから来たデータを時間差で下流に流すという挙動になるでしょう。しかし、Recoilの場合はどこかで非同期処理が始まった時点で、下流を全部同時にサスペンドさせます。こうしないと、そこで非同期処理の前後の時点の状態が共存してしまい、一貫性が失われるからです。

以上の話題を図で表現してみました。次の図は**Recoil**における状態変化時の様子を示したものです。

![Recoilではグラフの途中のselectorが非同期処理を開始したら下流が全部サスペンドすることを示した図](/images/recoil-vs-rxjs/recoil-graph.png)

この図は頂点のatomとそれに連なるselectorたちを表示しています。最初はグラフ全体がAから計算された状態になっています。

頂点のatomがBに変化すると、それに依存するselectorの値も再計算されますが、そのうちの1つが非同期selectorだった場合を想定してみましょう。この場合、Recoilではそのselectorから下流が全部サスペンド（計算中）状態になります。そして、非同期処理が完了したらB由来の結果が下流にも流れていきます。

一方で、**RxJS**で素朴に実装すると次のようになることが想定されます。

![RxJSによる素朴な実装では、グラフの途中に非同期処理があったら下流に変更が伝わらないことを示した図](/images/recoil-vs-rxjs/rxjs-graph.png)

この図では、グラフの途中に非同期処理があった場合、その結果がイベントとして下流に流れるのは非同期処理が完了してからです。そのため、その間はグラフの上流と下流で、A由来の状態とB由来の状態が混在しています。

Recoilのような挙動をRxJSで再現しようとすると、サスペンドイベントを別途流し、データフローグラフを構成するoperatorたちが全部そのプロトコルを知っていなければならないなど、結局Recoilの再実装になることが想定されます。

このように、Recoilでは非同期処理が混ざったロジックにおいても、複数時点の状態が混ざることがなく、データフローグラフの全体がひとつの特定の状態に由来するようになっています。これがRecoilが提供するデータの一貫性です。

ということで、以下ではこの話を具体的なサンプルを交えて説明します。

# サンプルに見るRecoilとRxJSの共通点・相違点

今回のサンプルアプリはこれです。

https://github.com/uhyo/recoil-vs-rxjs-example

![アプリのスクリーンショット](/images/recoil-vs-rxjs/sample-screenshot.png)

今回はRecoilとRxJSで同じようなものを実装しており、コードを比較できるようになっています。「第1世代」〜「第8世代」の選択肢が用意されており、選択するとその下に表示されているリストの内容が外部から取得されて表示されるというものです。データは例によって[PokéAPI](https://pokeapi.co/)から取得しています。

## Recoilの実装

それぞれの実装を見てみましょう。Recoilの実装はこのようになっています。

:::details Recoilの実装の全体

```ts
export const generationState = atom({
  key: "generation",
  default: 1,
});

const client = new Client({
  url: "https://beta.pokeapi.co/graphql/v1beta",
});

type Pokemon = {
  id: number;
  name: string;
};

const pokemonListQuery = gql`
  query ($generation: Int!) {
    species: pokemon_v2_pokemonspecies(
      where: { generation_id: { _eq: $generation } }
      order_by: { id: asc }
    ) {
      id
      pokemon_v2_pokemonspeciesnames(where: { language_id: { _in: [11] } }) {
        language_id
        name
      }
    }
  }
`;

export const pokemonListState = selector<readonly Pokemon[]>({
  key: "pokemonList",
  async get({ get }) {
    const generation = get(generationState);

    const response = await client
      .query(pokemonListQuery, {
        generation,
      })
      .toPromise();
    if (response.error) {
      throw response.error;
    }
    if (!response.data) {
      throw new Error("No Data");
    }
    return response.data.species.map((s: any) => ({
      id: s.id,
      name: s.pokemon_v2_pokemonspeciesnames[0].name,
    }));
  },
});
```

:::

筆者の他の記事を読んだ方にとってはおなじみの、何の変哲もない実装です。

世代選択の部分は、ユーザーが好きに選択できるのでatomで表現されています。

```ts
export const generationState = atom({
  key: "generation",
  default: 1,
});
```

そして、リストはこのatomに依存する非同期selectorとして表現されています。

```ts
export const pokemonListState = selector<readonly Pokemon[]>({
  key: "pokemonList",
  async get({ get }) {
    const generation = get(generationState);

    const response = await client
      .query(pokemonListQuery, {
        generation,
      })
      .toPromise();
    if (response.error) {
      throw response.error;
    }
    if (!response.data) {
      throw new Error("No Data");
    }
    return response.data.species.map((s: any) => ({
      id: s.id,
      name: s.pokemon_v2_pokemonspeciesnames[0].name,
    }));
  },
});
```

## RxJSの実装

次に、RxJSの実装です。

:::details RxJSの実装の全体

```ts
export const generationSubject = new BehaviorSubject(1);

export const pokemonList = generationSubject.pipe(
  mergeMap((generation) => requestPokemonList(generation)),
  share()
);

const client = new Client({
  url: "https://beta.pokeapi.co/graphql/v1beta",
});

type Pokemon = {
  id: number;
  name: string;
};

const pokemonListQuery = gql`
  query ($generation: Int!) {
    species: pokemon_v2_pokemonspecies(
      where: { generation_id: { _eq: $generation } }
      order_by: { id: asc }
    ) {
      id
      pokemon_v2_pokemonspeciesnames(where: { language_id: { _in: [11] } }) {
        language_id
        name
      }
    }
  }
`;

function requestPokemonList(
  generation: number
): Observable<readonly Pokemon[]> {
  return new Observable<any>((subscriber) => {
    const source = client.query(pokemonListQuery, {
      generation,
    });
    source((signal) => {
      if (signal === 0 /*SignalKind.End */) {
        subscriber.complete();
        return;
      }
      if (signal.tag === (1 as SignalKind.Push)) {
        if (signal[0].error) {
          subscriber.error(signal[0].error);
        }
        subscriber.next(signal[0].data);
      }
    });
  }).pipe(
    map((data): readonly Pokemon[] => {
      return data.species.map((s: any) => ({
        id: s.id,
        name: s.pokemon_v2_pokemonspeciesnames[0].name,
      }));
    })
  );
}
```

:::

RxJSを日常的に使用しているわけではないので完璧な実装かどうか分かりかねますが、素直に書くとこうだろうと思われる実装になっています（もしイディオムから乖離していたらぜひコメントでご指摘ください）。

まず、世代選択については値を保持しておく必要があるので、`BehaviorSubject`での実装です。

```ts
export const generationSubject = new BehaviorSubject(1);
```

そして、この値が変化したら（`generationSubject`からイベントが流れてきたら）非同期処理を行なって結果のリストを流すObservableは次のように書けます。

```ts
export const pokemonList = generationSubject.pipe(
  mergeMap((generation) => requestPokemonList(generation)),
  share()
);
```

ちなみに、`requestPokemonList`は新しいObservableを作って返す実装です。

```ts
function requestPokemonList(
  generation: number
): Observable<readonly Pokemon[]> {
  return new Observable<any>((subscriber) => {
    const source = client.query(pokemonListQuery, {
      generation,
    });
    source((signal) => {
      if (signal === 0 /*SignalKind.End */) {
        subscriber.complete();
        return;
      }
      if (signal.tag === (1 as SignalKind.Push)) {
        if (signal[0].error) {
          subscriber.error(signal[0].error);
        }
        subscriber.next(signal[0].data);
      }
    });
  }).pipe(
    map((data): readonly Pokemon[] => {
      return data.species.map((s: any) => ({
        id: s.id,
        name: s.pokemon_v2_pokemonspeciesnames[0].name,
      }));
    })
  );
}
```

RxJSとReactの繋ぎ込みはこういうシンプルな感じにしました。

```ts
export const useObservable = <T>(observable: Observable<T>): T | undefined => {
  const [current, setCurrent] = useState<T | undefined>(undefined);
  useEffect(() => {
    const subscription = observable.subscribe({
      next: (value) => {
        setCurrent(value);
      },
      error: (error) => {
        throw error;
      },
    });
    return () => {
      subscription.unsubscribe();
    };
  }, [observable]);
  return current;
};
```

## 両者の挙動の違い

RecoilとRxJSの実装、どちらも非同期処理を交えたロジックの記述という点では素直な実装になっていますが、両方の実装には挙動の違いがあります。それは、この記事ですでに説明したとおりです。

具体的には、選択されている世代を変更したときの挙動です。

Recoilの方は、世代を変更したのと同時にリストが「Loading...」表示になります。新しいデータが届いたらそのデータが表示されます。

一方で、RxJSでは世代を変更しても少しの間古いリストが残っています。そして、新しいリストが取得されたらそれに切り替わります。

RxJSのこの挙動は、少しの間古い状態（世代を変える前の状態）と新しい状態（世代を変えた後の状態）が混在した状況がUIに表出してしまっていると言えます。次の図（再掲）の真ん中にある、A由来の状態とB由来の状態がデータフローグラフの中で混在してしまっている状態です。

![RxJSによる素朴な実装では、グラフの途中に非同期処理があったら下流に変更が伝わらないことを示した図](/images/recoil-vs-rxjs/rxjs-graph.png)

もちろん、RxJSでもうまく実装すればRecoilと同じ挙動をするように実装できるでしょうが、先述のようにそれ用のイベントのフォーマットを作ったりなど追加の手当てが必要です。Recoilはそれを内蔵しているからこそ、シンプルな実装で一貫性のある挙動を得ることができています。

# まとめ

いかがでしたか？

この記事では、非同期なステート計算を書けるという点に着目してRecoilとRxJSを比較しました。

同じくらい素朴な実装で比較すると、Recoilはライブラリにステートの一貫性保証が組み込まれているという点で両者に差が出ていることが明らかになりました。