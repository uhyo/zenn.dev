---
title: "非制御フォームをやるならこんなふうに　Recoil編"
emoji: "🐒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "recoil"]
published: true
---

Reactにおいて、フォームをどのように実装するかというのは開発者の悩みの種のようです。筆者は最近ロジックをRecoilに載せるのにはまっていますので、今回はRecoilを使ってフォームを実装することを考えてみます。

# 制御コンポーネントと非制御コンポーネント

Reactにおいてフォームの実装方法は2種類に大別されます。それは、**制御コンポーネント** (controlled components) を使うか**非制御コンポーネント** (uncontrolled components) を使うかです。制御コンポーネントとは、入力されたテキスト等をReactのステートとして保持し、`<input value={state} />`のようにinput等のvalueに渡してレンダリングする方法です。制御コンポーネントではデータの本体がReact側にあり、DOMはそれを写像しているだけです。一方、非制御コンポーネントの場合、`<input>`に入力されている状態はReactの状態としては保持されていません。必要になったときに生DOMを参照して値を取得することになります。

何となく制御コンポーネントのほうが世の中の主流のように見えますが、今回は敢えて**非制御コンポーネント**で挑みます。その理由は以下の3つです。

- 制御コンポーネントはユーザーがタイピングするごとに再レンダリングするので無駄がある気がする。
- もともとDOM側でユーザーの入力状態を管理してくれる仕組みなので、React側で管理するのは二重管理になっている。
- というか、仕組み上ユーザーが先にDOMを変更してからReactのステートに反映しているのでそもそも微妙な仕組みだと感じる。

一方で、非制御コンポーネントを用いる場合、次の点が課題になります。

- ユーザーの入力に応じてインタラクティブにバリデーション等のフィードバックを返すのが難しい（できるが、きれいに書きにくい）。

そこで、今回の記事では**Recoil**を用いてこの点を克服することを目指します。

一言で言えば、**フォームの入力内容をAtom Effectsを用いてRecoilステートに同期しよう**ということです。

# Recoilで非同期コンポーネントをやる方法

筆者はロジックをRecoilに載せたいと考えていますので、フォームに対するバリデーションも当然Recoilでやります（理想的にはHTMLに組み込みの機能でできればば良いですが、ユーザーにいい感じのフィードバックを返すことを考えるとやはり限界があります）。

そうなると、Recoilの外 (DOM) で管理されているデータをRecoilデータフローグラフの中に持ち込む必要があります。Recoilでは、このような目的のためにatom effectsが有用です。こうすると、ユーザーの入力状態をatomとして得ることができます。[筆者の以前のトーク](https://speakerdeck.com/uhyo/sutetoguan-li-wochao-erurecoilyun-yong-nokao-efang)でも説明したように、atomというのはコアな状態を表すために使うものですが、atom effectsを使う場合はそうではないこともあります。今回の場合は、コアな状態（ユーザーの入力）はDOMの中にあり、それをRecoilの世界に反映したものがatomになります。

ユーザーの入力をatomに反映すれば、それに対するバリデーションロジックはselectorとして記述することができます。それをReact層に渡すことで、バリデーション結果が変化したときにだけ再レンダリングを行うことができます。

# まずはやってみる

ここからはコードを用いて説明していきます。リポジトリはこちらです。

https://github.com/uhyo/recoil-uncontrolled-form

まずは、原始的な方法でやってみます。

## フォームの用意

とりあえずフォームを用意しましょう。コードとしてはこのような感じです。

```tsx
<form id="pokemon-form">
  <p>
    <label>
      好きなポケモンはなんですか？
      <input type="text" name="name" required />
    </label>
  </p>
  <hr />
  <p>
    <button type="submit">送信</button>
  </p>
</form>
```

今回フォームに`id`を与えているのは、Recoilにデータを取り込む際にDOMからIDで引っ張ろうという魂胆です。

## Atom Effectsを書く

では、上のフォームの内容が反映されたatomを作りましょう。一応、ちょっとだけ汎用的なatomとして`formContents`を用意しました。

:::details formContentsの実装

```ts
export const formContents: <Keys extends string>(input: {
  formId: string;
  elementNames: readonly Keys[];
}) => RecoilValueReadOnly<Record<Keys, string>> = atomFamily<
  Record<string, string>,
  {
    formId: string;
    elementNames: readonly string[];
  }
>({
  key: "dataflow/utils/formContents",
  effects: ({ formId, elementNames }) => [
    ({ setSelf, resetSelf }) => {
      type State = {
        form: HTMLFormElement | null;
        cleanup: () => void;
      };
      let state: State = {
        form: null,
        cleanup: () => {},
      };
      queryAndSet();
      const observer = new MutationObserver(() => {
        queryAndSet();
      });
      observer.observe(document, {
        childList: true,
        subtree: true,
      });
      return () => {
        state.cleanup();
        observer.disconnect();
      };
      function queryAndSet() {
        const formElm = document.getElementById(formId);
        if (formElm === state.form) {
          return;
        }
        state.cleanup();

        if (formElm instanceof HTMLFormElement) {
          const obj: Record<string, string | null> = Object.fromEntries(
            elementNames.map((key) => {
              const control = formElm.elements.namedItem(key);
              if (control instanceof HTMLInputElement) {
                return [key, control.value];
              } else {
                return [key, null];
              }
            })
          );
          if (Object.values(obj).some((value) => value === null)) {
            resetSelf();
            return;
          }
          setSelf(obj as Record<string, string>);
          const inputHandler = (e: Event) => {
            const target = e.target;
            if (!(target instanceof HTMLInputElement)) {
              return;
            }
            const name = target.name;
            const value = target.value;
            setSelf((current) => {
              if (current instanceof DefaultValue) {
                return current;
              }
              if (current[name] === value) {
                return current;
              }
              return {
                ...current,
                [name]: value,
              };
            });
          };
          formElm.addEventListener("input", inputHandler);
          state = {
            form: formElm,
            cleanup: () => {
              formElm.removeEventListener("input", inputHandler);
            },
          };
        } else {
          resetSelf();
        }
      }
    },
  ],
});
```

:::

長いのでコード全体は折りたたんでいます。やっていることを要点に絞って説明します。

この`formContents`はatom familyであり、引数として`formId`（フォームのDOM上のID）と`elementNames`（フォーム上で監視対象とする`name`の配列）を受け取ります。

atomが初期化されると、当該のform要素に対してinputイベントのハンドラを設定し、配下の要素に対する編集を監視、変更があればatomの内容として反映します。

また、それ以外に監視対象のform要素自体が書き換えられた場合などに備えて、MutationObserverを設定してformが変わった場合も追随するようにしています。

## フォームと接続する

そして、このutilを用いて先ほど作ったフォームと接続するには、次のようにします。

```ts
export const pokemonForm = formContents({
  formId: "pokemon-form",
  elementNames: ["name"],
});
```

この`pokemonForm`はatomであり、ユーザーの入力に応じて`{ name: "ピカチュウ" }`のような値をとります。

ちなみに、このような実装だとReact側で`<input name="name">`を書き忘れるといったミスを起こしてしまいそうですが、今のところその場合は`pokemonForm`がサスペンドしたままになり、型のミスマッチはとりあえず起こらないようになっています。

## バリデーションを書く

データがRecoilに載れば、あとはいつものやり方でバリデーションを書くだけです。今回は、入力された名前が存在しているかどうかチェックするようにしましょう。そのための実装は次のようになります。

```ts
export const validationState = selector<ValidationState>({
  key: "dataflow/validation",
  get({ get }) {
    const formData = get(pokemonForm);
    const pokemonList = get(pokemonListState);

    const pokemonNameIsValid = pokemonList.some(
      (p) => p.name === formData.name
    );
    const canSubmit = pokemonNameIsValid;

    return {
      pokemonNameIsValid,
      canSubmit,
    };
  },
});
```

ちなみに、`pokemonListState`は[前回の記事](https://zenn.dev/uhyo/articles/recoil-selector-infinite-scroll)に引き続き[PokéAPI](https://pokeapi.co/)からデータを取得するようになっています。

## バリデーションをReact層に接続する

バリデーションのロジックができたら、Reactコンポーネントから利用します。例えば、バリデーションが通っていない場合は送信ボタンがdisabledになるようにするには、次のようにします（バリデーションが通っていない場合に送信ボタンをdisabledにするのは良くないという記事を見た気もしますが、別の話題なので今回は気にしません）。

```tsx
export const SubmitButton: FC = () => {
  const canSubmit = useRecoilValue(field(validationState, "canSubmit"));
  return (
    <button type="submit" disabled={!canSubmit}>
      送信
    </button>
  );
};
```

ちなみに、ここで軽く登場している `field` は次のように定義されるユーティリティで、ステートオブジェクト全体のうち特定のプロパティだけをサブスクライブするためのものです。上の例では、`validationState`が変わっても、その`canSubmit`プロパティの値が変わらない限り`SubmitButton`は再レンダリングされません。

:::details fieldの定義

```ts
const _field = selectorFamily<
  unknown,
  {
    selector: RecoilValueReadOnly<Record<PropertyKey, unknown>>;
    key: PropertyKey;
  }
>({
  key: "dataflow/utils/field",
  get:
    ({ selector, key }) =>
    ({ get }) =>
      get(selector)[key],
});

export const field: <T, K extends keyof T>(
  selector: RecoilValueReadOnly<T>,
  key: K
) => RecoilValueReadOnly<T[K]> = (selector, key) =>
  _field({
    selector: selector as RecoilValueReadOnly<any>,
    key,
  }) as RecoilValueReadOnly<any>;
```

:::

もう一つバリデーションロジックは`pokemonNameIsValid`も用意していましたので、こちらもReactに接続します。

```tsx
export const PokemonNameValidationStatus: FC = () => {
  const pokemonNameIsValid = useRecoilValue(
    field(validationState, "pokemonNameIsValid")
  );
  if (pokemonNameIsValid) {
    return (
      <p style={{ color: "green", fontSize: "0.8em" }}>
        ポケモンの名前を入力してください ✅
      </p>
    );
  } else {
    return (
      <p style={{ color: "gray", fontSize: "0.8em" }}>
        ポケモンの名前を入力してください
      </p>
    );
  }
};
```

そして、これを最初のフォームに接続すれば実装完了です。

```tsx
<form id="pokemon-form">
  <p>
    <label>
      好きなポケモンはなんですか？
      <input type="text" name="name" required />
    </label>
  </p>
  <Suspense>
    <PokemonNameValidationStatus />
  </Suspense>
  <hr />
  <p>
    <Suspense>
      <SubmitButton />
    </Suspense>
  </p>
</form>
```

## 送信時のデータ取得

ちなみに、フォームが送信されたときはユーザーが入力したデータを取得する必要があります。そのためには、useRecoilCallbackを使用してatomからデータを得るのが適切です。

```ts
const submitHandler = useRecoilCallback(
  ({ snapshot }) =>
    (e: React.SyntheticEvent<HTMLFormElement>) => {
      snapshot.getPromise(pokemonForm).then((contents) => {
        alert(contents.name);
      });
      e.preventDefault();
    },
  []
);
```

以上のコードは[こちらのコミット](https://github.com/uhyo/recoil-uncontrolled-form/tree/e2caa71f95152c99581cc076e29c70428ca4fd0e)に対応しています。

## スクリーンショット

![「フシギ」と入力し、バリデーションが通っていない場合の表示](/images/recoil-uncontrolled-form/sample-invalid.png)

![「フシギダネ」と入力し、バリデーションが通っていない場合の表示](/images/recoil-uncontrolled-form/sample-valid.png)

以上の実装で、このように非制御コンポーネントかつリアルタイムのフィードバックを行うフォームが実装できました。

## Recoilを用いる非同期コンポーネントのアーキテクチャ

今回の実装のアーキテクチャを図にしてみました。

![アーキテクチャ図](/images/recoil-uncontrolled-form/architecture.png)

繰り返しになりますが、ポイントは、フォームが保持する状態の本体はあくまでDOMの中にあり、それを同期してRecoilのatomに反映している点です。そして、そこからselectorを展開して必要な計算を行います。

以上のような実装をすると、ユーザーが入力するごとに再レンダリングが発生するのではなく、バリデーション結果が変化して表示を変える必要がある場合のみ再レンダリングが行われるようになるため何となく嬉しいですね。

今回の実装は、ユーザーが入力するたびに再レンダリングが発生しないという点では非制御コンポーネントと同じですが、状態をRecoilデータフローグラフに持ち込むことにより、制御コンポーネントの場合と同様の扱いやすさ（バリデーションなどの書きやすさ）を実現しています。

# インターフェースを整える

上の実装はフォームのIDに依存しているなど、やや小慣れていない実装でした。そこで、将来のライブラリ化を見据えてもう少しインターフェースを整えてみました。コードは[こちらのコミット](https://github.com/uhyo/recoil-uncontrolled-form/tree/fec3e7d335e9d7598a413e76ade65adfdbb61cf99)にあります。

今回は、まずフォームを最初に定義します。そのために`recoilForm`という関数が用意されています。

```ts
export const pokemonForm = recoilForm({
  elements: ["name"],
});
```

そして、ここで定義したフォームは次のようにレンダリングします。

```tsx
export const PokemonForm: FC = () => {
  const submitHandler = useRecoilCallback(
    ({ snapshot }) =>
      (e: React.SyntheticEvent<HTMLFormElement>) => {
        snapshot.getPromise(formData(pokemonForm)).then((contents) => {
          alert(contents.name);
        });
        e.preventDefault();
      },
    []
  );

  const pokemonList = useRecoilValue(pokemonListState);

  const { Form, Input } = useRecoilValue(pokemonForm);

  return (
    <Form onSubmit={submitHandler}>
      <p>
        <label>
          好きなポケモンはなんですか？
          <Input type="text" name="name" required list="pokemon-list" />
          <datalist id="pokemon-list">
            {pokemonList.map((poke) => (
              <option key={poke.id}>{poke.name}</option>
            ))}
          </datalist>
        </label>
      </p>
      <Suspense>
        <PokemonNameValidationStatus />
      </Suspense>
      <hr />
      <p>
        <Suspense>
          <SubmitButton />
        </Suspense>
      </p>
    </Form>
  );
};
```

ポイントは、`pokemonForm`の中身を`useRecoilValue`で取得すると、`Form`や`Input`といったコンポーネントが得られる点です。これを普通の`<form>`や`<input>`の代わりに使用します。こうすることで、Recoilの内部で自動的に入力内容がトラッキングされるようになります。

フォームの入力内容に対するバリデーションはこんな感じです。

```ts
export const validationState = selector<ValidationState>({
  key: "dataflow/validation",
  get({ get }) {
    const formContents = get(formData(pokemonForm));
    const pokemonList = get(pokemonListState);

    const pokemonNameIsValid = pokemonList.some(
      (p) => p.name === formContents.name
    );
    const canSubmit = pokemonNameIsValid;

    return {
      pokemonNameIsValid,
      canSubmit,
    };
  },
});
```

ポイントは、`formData`という別の関数を使って`get(formData(pokemonForm))`のようにすることで、ユーザーがフォームに入力した値を得ることができます。あとは先ほどと同様にバリデーションを行うだけです。

インターフェースを整えたことで、`<form>`にidを付与する必要がなくなって少しきれいなインターフェースになりましたね。

# まとめ

ということで、今回はReactのフォームを非同期コンポーネントとRecoilで実装する方法を紹介しました。個人的には、同期コンポーネントの微妙に感じる部分を直しつつ、使い勝手も悪くない方法ではないかと思います。

なお、今回のコードは`<input>`にしか対応していないなど最低限のPoCなので、実際に使用する際はご注意ください。このようなやり方の評判がよければちゃんとライブラリを整備するかもしれません。

