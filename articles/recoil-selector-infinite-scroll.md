---
title: "Recoil selector活用パターン 無限スクロール編"
emoji: "📜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "recoil"]
published: true
---

みなさんこんにちは。筆者は最近Recoilを使ってロジックを記述するのにハマっています。先日はそのようなテーマでトークをしましたので、よければご覧ください。

@[speakerdeck](cf38bb95ed904eb6a97b5671f03cdd3e)

要点は、 **Recoilのselectorとかも活用しまくってロジックをどんどんRecoilに載せようぜ！！** ということです。ただ、前記のイベントを観ていただいた方には分かるように、Recoilを活用していてもほとんどatomしか使っていないという場合もあり、Recoilの普及度とselectorの普及度には差があるようです。

そこで、この記事ではRecoil selectorの活用パターンを紹介します。今回は**無限スクロール**の実装です。

ここで想定している無限スクロールとは、次のようなものです。

- サーバーから取得したデータがリストで表示されている。
- ユーザーがスクロールしてリストの下に到達したら、サーバーから追加のデータを読み込んでリストの下に継ぎ足す。

皆さんならこれをどう実装するでしょうか？　ぜひ考えてみてください。

# サンプルアプリケーション

この記事で使用するサンプルはこちらのリポジトリにあります。Viteなので`npm run dev`で動作させることができます。

https://github.com/uhyo/recoil-infinite-scroll-sample

次の画像はアプリのスクリーンショットです。今回はリストの提供元として[PokéAPI](https://pokeapi.co/)を使用させていただき、ポケモンのリストを読み込んで表示するようにしました。

![アプリのスクリーンショット](/images/recoil-selector-infinite-scroll/app-screenshot.png)

# 実装方針

Reactを使っている方の中には、無限スクロールの実装を経験したことがある方も結構いるのではないでしょうか。典型的な設計は次のようなものです。

- 読み込まれたリストを`useState`とかで持っておく。
- ユーザーがリストの下端にたどり着いたら追加の読み込みを発火し、読み込まれたらステートを更新してステートを継ぎ足す。

しかし、今回はその方針を採りたくありません。その理由は、上述のスライドに書いてあります。**コアな状態のみをステート (atom) として扱い、データの取得結果などはatomではなくselectorにしたい**からです。

ということで、読み込まれたリストの本体は意地でもatomにせずにselectorに載せたいと思います。では、atomにすべき状態とは何でしょうか。これをパッと思いついた方はなかなかRecoilにロジックを載せる資質があると思います。

答えは、**表示すべき要素数**です。

![atom（要素数）→selector（リスト）](/images/recoil-selector-infinite-scroll/graph1.png)

上の画像では、丸がatomで長方形がselectorです。つまり、要素数のatomが最初が50であればselectorは50件を読み込んで返します。Recoilでは非同期selectorが可能なので、APIからデータを取得するというロジックはselectorで書くことができます。

ユーザーがリストの下端についた時は、リストの要素を増やす必要があるため、要素数のatomを更新します。リストのselectorがそれに反応して、要素が増やされたリストを返します。

この記事では、atomは要素数のみ、他は全部selectorで実装します。ということで、この「リスト」のselectorをどう実装するかが鍵になります。

# 望ましくない実装

上の図を要件として見せられたら、まず思いつくのは次のような実装でしょう。

```ts
export const pokemonListQuery = selector<
  QueryResult
>({
  key: "dataflow/pokemonListQuery",
  get:
    async ({ get }) => {
      // 必要な要素数を取得
      const limit = get(totalCount);
      // サーバーからデータを取得
      const result = await client
        .query(query, {
          offset: 0,
          limit,
        })
        .toPromise();
      if (result.error) {
        throw result.error;
      }
      if (result.data === undefined) {
        throw new Error("No data");
      }
      return result.data;
    },
});
```

重要な部分以外は端折っていますが、要するにselectorのgetの中でAPI呼び出しを行うだけです（今回使用するAPIはGraphQLなのでクライアントとして[urql](https://formidable.com/open-source/urql/)を使用しています）。その際、何件必要かを要素数のatom（`totalCount`）から取得し、それをパラメータの`limit`に渡しています。

しかし、このような実装は望ましくありません。とくに、要素数が増えた場合に、すでに読み込まれていた部分も全件取得て全件取得してしまうので無駄があります。理想的には、要素数が増えた場合は差分だけ読み込みたいですね。

# 再帰selectorによる解決

ということで、次は望ましい実装をselectorで実現する方法を考えます。そのために使うテクニックが、**再帰selector**です（正確にはselectorFamilyですが）。

## アイデアの説明

再帰selectorと言われてもピンと来ないかもしれませんので、一旦Recoilのことは忘れて普通の関数で説明します。ページングが必要なAPIから要求されただけデータを取得して返す関数は、普通に書くとこのようになるでしょう。

```ts
async function getListFromAPI(totalItems: number): Data[] {
  let result: Data[] = [];
  while (result.length < totalItems) {
    const chunk = await loadFromAPI({
      offset: result.length,
      limit: pageSize,
    });
    result = result.concat(chunk);
  }
  return result;
}
```

ところが、関数型言語だと`let`が無かったり配列を破壊的変更できなかったりします。その場合は、上のような実装はできません。

そして、Recoilのselectorを書くときの環境も実際このような状態です。破壊的変更ができないのは言わずもがな、`let`が無いというのは「selectorの値を計算するときにselectorの以前の値を利用することができない」という制約に相当します。そのため、上の実装をそのままRecoilのselectorに治すことはできません。

では、どうすれば良いでしょうか。実は、再帰関数を使えば次のように実装できます。

```ts
async function getListFromAPIRec(totalItems: number, offset: number): Data[] {
  const chunk = await loadFromAPI({
    offset,
    limit: pageSize,
  })
  if (offset + chunk.length >= totalItems) {
    return chunk;
  }
  const rest = await getListFromAPIRec(totalItems, offset + chunk.length);
  return chunk.concat(rest);
}

function getListFromAPI(totalItems: number): Data[] {
  return getListFromAPIRec(totalItems, 0);
}
```

つまり、`getListFromAPIRec`の1回の呼び出しで1ページを取得し、まだデータが足りなければoffsetをずらして再帰呼び出しします。このようにすることで、同じ処理がイミュータブルに実装できました。

この実装ならば、Recoilでも再現できます。そうすればselectorだけで無限スクロールが実装できます。図にするとこのようになります。

![リストselectorが裏で再帰selectorを呼び出し、再帰selectorは3回再帰している。再帰selectorは裏でurql selectorを呼び出している](/images/recoil-selector-infinite-scroll/graph2.png)

上の図にある再帰selectorは、引数として `totalItems` と `offset` を受け取ります。上のサンプルコードと同じですね。そして、引数を `offset` と `limit` に変換して、GraphQL呼び出しを担当するurql selectorを呼び出します。ポイントは、 `totalItems`が更新されると再帰selectorの引数も変わる（＝再帰selectorの値が計算され直す）が、urql selectorに渡される引数は変わらないということです。これにより、`totalItems`が更新された際も、すでに読み込まれた部分は再読み込みされません。再帰selectorの役割は、このような引数の変換、そして再帰条件の判定（十分なデータが読み込まれたら再帰しない）です。

## コードで見る

ということで、この実装を実際のRecoilのコードで見てみましょう。まず、urql selectorは前述のselectorのコードを少し書き換えて、offsetとlimitを引数として受け取るselectorFamilyにすればできます。

```ts
export const pokemonListQuery = selectorFamily<
  QueryResult,
  {
    limit: number;
    offset: number;
  }
>({
  key: "dataflow/pokemonListQuery",
  get:
    ({ limit, offset }) =>
    async () => {
      const result = await client
        .query(query, {
          offset,
          limit,
        })
        .toPromise();
      if (result.error) {
        throw result.error;
      }
      if (result.data === undefined) {
        throw new Error("No data");
      }
      return result.data;
    },
});
```

そして、再帰selectorはこうです（少しずつ解説するのでコード全体は折りたたんでおきます）。

:::details 再帰selectorのコード

```ts
const pokemonListRec = selectorFamily<
  PokemonListState,
  {
    requestedItems: number;
    offset: number;
  }
>({
  key: "dataflow/pokemonList/pokemonListRec",
  get:
    ({ requestedItems, offset }) =>
    ({ get }): PokemonListState => {
      const limit = Math.min(requestedItems - offset, pageSize);
      const pokemons = get(
        formattedPokemonListQuery({
          limit,
          offset,
        })
      );

      if (pokemons.length < limit) {
        return {
          pokemons,
          mightHaveMore: false,
        };
      }
      if (requestedItems === offset + limit) {
        return {
          pokemons,
          mightHaveMore: true,
        };
      }
      const rest = get(
        noWait(
          pokemonListRec({
            requestedItems,
            offset: offset + limit,
          })
        )
      );
      switch (rest.state) {
        case "hasError": {
          throw rest.errorMaybe();
        }
        case "loading": {
          return {
            pokemons,
            mightHaveMore: true,
          };
        }
        case "hasValue": {
          return {
            pokemons: [...pokemons, ...rest.contents.pokemons],
            mightHaveMore: rest.contents.mightHaveMore,
          };
        }
      }
    },
});
```

:::

再帰selectorの引数は `requestedItems` と `offset` です。つまり、このselectorの責務は「全部で `requestedItems` 個のデータのうち、 `offset` 番目以降のデータを全部読み込む」ことです。

まず、自身がどれくらいデータを読み込むか計算します。基本的には `pageSize`（定数）個読み込めばよいですが、端数の処理を考えてこのようになります。

```ts
const limit = Math.min(requestedItems - offset, pageSize);
const pokemons = get(
  formattedPokemonListQuery({
    limit,
    offset,
  })
);
```

そして、再帰する必要があるかどうか判断します。なければ、結果をそのまま返します。

```ts
if (pokemons.length < limit) {
  return {
    pokemons,
    mightHaveMore: false,
  };
}
if (requestedItems === offset + limit) {
  return {
    pokemons,
    mightHaveMore: true,
  };
}
```

今回結果がリストだけでなく`mightHaveMore`というフラグが追加されています。このフラグがfalseになったらリストを全件読み込み終わったという意味になります。今回は、`limit`よりも少ないデータが返ってきた場合は全件読み込んだと判断します。1つ目のif文がこの場合です。

2つ目のif文は、今回読み込んだデータで以って要求数の読み込みが完了した場合の処理です。

これらの条件を満たさなかった場合、今回の読み込みだけではデータが足りていないので、再帰して続きを読み込んでもらいます。その処理がこうです。

```ts
const rest = get(
  noWait(
    pokemonListRec({
      requestedItems,
      offset: offset + limit,
    })
  )
);
switch (rest.state) {
  case "hasError": {
    throw rest.errorMaybe();
  }
  case "loading": {
    return {
      pokemons,
      mightHaveMore: true,
    };
  }
  case "hasValue": {
    return {
      pokemons: [...pokemons, ...rest.contents.pokemons],
      mightHaveMore: rest.contents.mightHaveMore,
    };
  }
}
```

今回、再帰の際に結果を`noWait`でラップしています。これはRecoilのutil的なselectorFamilyで、与えられたselectorの結果をLoadableとして取得します。こうすると、再帰先がサスペンドした場合でも自身はサスペンドしなくなります。

今回これを使用している理由は、リストの追加読み込み中にサスペンドさせたくないからです。理想的にはReactのトランジション機能を使用すべきですが、残念ながらRecoilのトランジション対応はunstableとされていえるため、今回はこちらの方法を使用しました。

残りのデータが読み込み中（`"loading"`）の場合はとりあえず今あるデータを返します（ローディング中かどうかをUIに出す必要がある場合は `loading` みたいなフラグを追加すればよいでしょう）。残りのデータがある場合は、自身のデータと結合して返します。

## 追加ロードの発火側のコード

ついでに、追加ロードを発火する側のコードを見ておきましょう。

:::details Loadingコンポーネント

```tsx
export const Loading: FC = () => {
  const observedRef = useRef<HTMLParagraphElement | null>(null);
  const { loadNextPage } = usePaging();

  useEffect(() => {
    if (observedRef.current === null) {
      return undefined;
    }
    let lastTriggerTime = 0;
    const observer = new IntersectionObserver(
      (entries) => {
        for (const entry of entries) {
          if (entry.isIntersecting) {
            if (lastTriggerTime + 1000 <= Date.now()) {
              lastTriggerTime = Date.now();
              loadNextPage();
            }
          }
        }
      },
      {
        rootMargin: "0px 0px 100px 0px",
      }
    );
    observer.observe(observedRef.current);
    return () => {
      observer.disconnect();
    };
  }, [loadNextPage]);

  return <p ref={observedRef}>Loading...</p>;
};
```

:::

コードを全部見る必要はありませんので畳んでおきましたが、要するにユーザーがこのコンポーネントに近づいたら`usePaging()`から取得した `loadNextPage`を呼び出します。その`usePaging`の実装はこうです。

```ts
export const usePaging = () => {
  const loadNextPage = useRecoilCallback(
    ({ set }) =>
      () => {
        set(totalItems, (count) => count + pageSize);
      },
    []
  );
  return {
    loadNextPage,
  };
};
```

記事の序盤で述べた通り、要素数（`totalItems`）を増やすだけです。とても宣言的で嬉しいですね。

# まとめ

この記事では、Recoilにおいてコアな状態だけをatomにするという設計を保ちながら、無限スクロールを実装する方法を紹介しました。

ポイントは、普通にやるとミューテーションが必要になる（＝atomが必要になる）ところを再帰selectorによって解決している点です。再帰selectorはロジックをselectorに載せるにあたって有用なテクニックですから、ぜひ使ってみてください。

## Q&A

**Q.** 再帰selectorとか正気か？

**A.** Twitterで「再帰関数とか正気か？」と言ってみてください。多分燃えますよ。