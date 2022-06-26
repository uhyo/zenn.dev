---
title: "ProviderタワーをRecoilに置き換える"
emoji: "🗼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

ReactアプリケーションではProviderタワーがよく見られます。Providerタワーは、アプリの上の方で次のコードのように複数のProviderが積み重なっている状態のことです（一般的な呼称かどうかは知りません）。

```tsx
const App: React.FC = () => {
  return (
    <FooProvider>
      <BarProvider>
        <BazProvider>
          <MainContents />
        </BazProvider>
      </BarProvider>
    </FooProvider>
  );
};
```

Providerは、コンテキストに対して値を供給する役割を担っており、コンポーネントツリー内でProviderより内側に配置されたコンポーネントからはそのコンテキストの値を参照することができます。コンテキストは、Reactにおいて外部ライブラリを使わずにステート管理（特にアプリ内の複数箇所で値を共有すること）を行うために使われます。

アプリケーションがだんだん発達してくると、コンテキストが増えてそれに対応するProviderがどんどん積み重なります。これがProviderタワーです。あまりにタワーが高層化すると、どこかのタイミングで他のステート管理ソリューションに移行したくなります。

この記事では、その場合の移行先としてRecoilが適していることを説明し、移行の具体例を示します。

# Providerタワーの何がまずいのか

コンテキストの使用量が増えることそれ自体は、必ずしも悪いことではありません。Providerがたくさん並んでいるのは見た目が悪い気もしますが、それ自体が問題というわけではありません。

問題となるのは、Providerの間に**依存関係**が生まれた場合です。具体例として簡単なアプリケーションを用意してみました。簡単なと言っても、複数のコンテキストが必要なのでi18nとモーダルダイアログを実装しています。

@[codesandbox](https://codesandbox.io/embed/provider-tower-6tldyu?fontsize=14&hidenavigation=1&theme=dark)

App.tsxを見てみると、3段のProviderタワーがあるのが分かります。

```tsx
export default function App(): JSX.Element {
  return (
    <Suspense fallback={null}>
      <LanguageSelectProvider>
        <I18nProvider>
          <ModalProvider>
            <MainContents />
          </ModalProvider>
        </I18nProvider>
      </LanguageSelectProvider>
    </Suspense>
  );
}
```

ポイントは、この3つのProviderはこの順番でなければならないということです。すなわち、`LanguageSelectProvider`を一番外側に置き、次に`I18nProvider`、そしてその内側に`ModalProvider`という順番である必要があります。これは、実装を見ていただけば分かりますが、`I18nProvider`のロジックが`LanguageSelectProvider`が提供するコンテキストに依存したりしているからです。

このように、あるコンテキストのロジックが他のコンテキストに依存するということは、Providerタワーが増設されるにつれて発生します。

ここに、コンテキストによる簡易的なステート管理の問題点があります。コンテキスト間の依存関係はこのようにコンポーネントツリー上で表現されなければならず、コンテキストの実装そのものではなくコンテキストの使用者側に委ねられています。これは、コンテキスト間の依存関係が、コンテキスト内でどの別のコンテキストから値を取り出しているかということを通じて、暗黙にしか表現されていないからです。

このように、コンテキスト同士の依存関係が複雑化して管理がやりにくくなったら、ステート管理を次の段階に進めるときです。

記事のタイトルにもあるように、筆者のおすすめは**Recoil**に移行することです。これは、Providerタワーによるステート管理と近いメンタルモデルで使うことができるので移行の障壁が低いことが理由です。また、次のポッドキャストで筆者が喋っているように、RecoilはReactのコンポーネントツリー外に**データフローグラフ**を構築することができます。これはちょうどProviderタワーの進化系として考えることができ、Providerタワーと似たようなモデルを採用しつつ、その間の依存関係を明確にすることができるのです。

https://uit-inside.linecorp.com/episode/123

この記事では、上のサンプルを利用して移行の具体例をお見せします。

# Recoilとデータフローグラフによるステート管理の提供

さっそくですが、サンプルをRecoilに移行したものがこちらです。

@[codesandbox](https://codesandbox.io/embed/provider-tower-to-recoil-1q7k4n?fontsize=14&hidenavigation=1&theme=dark)

`App`のコードを見ると、Providerタワーが解消されたことがわかります。`ModalProvider`だけはモーダル要素をレンダリングする役割があるので`ModalElements`として残しています。

```tsx
export default function App(): JSX.Element {
  return (
    <RecoilRoot>
      <Suspense fallback={null}>
        <MainContents />
        <ModalElements />
      </Suspense>
    </RecoilRoot>
  );
}
```

## ステートをatomで表現する

では、各コンテキストのbefore/afterを見てみましょう。元々一番上にあった`LanguageSelectProvider`は単純で、ユーザーが`ja`を選択しているか`en`を選択しているかを保持しているだけです。

```tsx
export type Language = "ja" | "en";

type LanguageSelectContextContent = {
  language: Language;
  setLanguage: (language: Language) => void;
};

const LanguageSelectContext = createContext<LanguageSelectContextContent>("ja");

export const LanguageSelectProvider: React.FC<{
  children: React.ReactNode;
}> = ({ children }) => {
  const [language, setLanguage] = useState<Language>("ja");

  const value: LanguageSelectContextContent = useMemo(() => {
    return {
      language,
      setLanguage
    };
  }, [language]);

  return (
    <LanguageSelectContext.Provider value={value}>
      {children}
    </LanguageSelectContext.Provider>
  );
};

export const useLanguageSelect = () => {
  return useContext(LanguageSelectContext);
};
```

`LanguageSelectProvider`の実装を見るとわかるように、このコンポーネントが`useState`でステートを保持し、自身の子孫たちに`LanguageSelectContext`を通じて情報を提供しています。この情報を使用したいコンポーネントのために`useLanguageSelect`フックを提供しています。

これをRecoilを用いて再実装すると次のようになりました。

```tsx
export type Language = "ja" | "en";

export const languageState = atom<Language>({
  key: "language",
  default: "ja"
});

export const useLanguageSelect = () => {
  const [value, setLanguage] = useRecoilState(languageState);
  return {
    language: value,
    setLanguage
  };
};
```

実装がずいぶんすっきりしましたね。従来の実装にボイラープレートが多く、これだけでもRecoilのありがたみが分かります。また、`useLanguageSelect`は同じインターフェースを保っているため、利用者側は実装を変更する必要がありません。このようにRecoilをラップするフックを提供するのが筆者のおすすめの使い方で、ProviderタワーからRecoilに移行する際の親和性も良好です。

ポイントは、従来`LanguageSelectProvider`の中に`useState`として存在していたステートが`atom`になったことです。このように、Recoilのデータフローグラフの中で状態を保持してほしいものはatomで表現します。

## 他のステートへの依存をselectorで表現する

次に、`I18nProvider`の以前の実装を見てみましょう。このProviderは、上に存在する`LanguageSelectProvider`に依存していました。

```tsx
type I18nData = Record<I18nKeys, string>;

type I18nContextValue = {
  data: I18nData;
};

const I18nContext = createContext<I18nContextValue>({
  get data(): never {
    throw new Error("I18nContext not initialized");
  }
});

const cachedData: Record<Language, I18nData | undefined> = {
  en: undefined,
  ja: undefined
};

export const I18nProvider: React.FC<{
  children: React.ReactChild;
}> = ({ children }) => {
  const { language } = useLanguageSelect();
  const data = cachedData[language];
  if (data === undefined) {
    throw import(`../i18n/${language}.json`).then((rawData) => {
      cachedData[language] = rawData;
    });
  }
  const value = useMemo(
    () => ({
      data
    }),
    [data]
  );

  return <I18nContext.Provider value={value}>{children}</I18nContext.Provider>;
};

export const useI18n = (key: I18nKeys) => {
  const { data } = useContext(I18nContext);
  return data[key];
};
```

この`I18nProvider`コンポーネントは、現在の言語を`useLanguageSelect`で取得し、それに応じた言語データ（`../i18n/${language}.json`）を読み込み、そのデータをコンテキストで下に提供する役割を持っています。React 18で導入されたSuspenseの機構もちゃっかり使っています（`throw import(...)`のところ）。

今回もこの`I18nProvider`が担っていたロジックをRecoilで書き換えます。すると次のようになりました。

```tsx
type I18nData = Record<I18nKeys, string>;

const cachedData: Record<Language, I18nData | undefined> = {
  en: undefined,
  ja: undefined
};

export const i18nState = selector<I18nData>({
  key: "i18n",
  get: async ({ get }) => {
    const language = get(languageState);
    const data = cachedData[language];
    if (data) {
      return data;
    }
    const rawData = await import(`../i18n/${language}.json`);
    cachedData[language] = rawData;
    return rawData;
  }
});

export const useI18n = (key: I18nKeys) => {
  const data = useRecoilValue(i18nState);
  return data[key];
};
```

このように、他のデータに依存するロジックはatomではなくselectorを用います。`get(languageState)`というところで、`i18nState`から`languageState`への依存を表現しています。

前の実装では`useLanguageSelect`を使っており、これはコンポーネントツリーの構成に依存してしまっていました。Recoilを用いた実装では、atomとselectorが直接明示的に接続されており、コンポーネントツリーに依存しません。これがRecoilを用いてデータフローグラフを構築することの、Providerタワーに比べた利点です。

残った`ModalProvider`については詳細な説明を省略しますが、こちらもatomやselectorを用いて表現できます。

# まとめ

この記事で具体例を交えて紹介したように、Recoilではatomやselectorを用いてデータフローグラフをReactのコンポーネントツリーから独立した形で構築することができます。これにより、コンテキストを利用するやり方において問題となっていた、コンテキスト間の依存関係が明瞭ではないという問題が解決されます。

Providerタワーを建設することは一番簡単なステート管理の方法として用いられていますが、それがつらくなった場合の自然な移行先としてRecoilがおすすめできます。この記事で見たようにコンテキストベースのインターフェースを変えずに中身をRecoilに置き換えることも可能ですから、Providerタワーを部分的に移行するような戦略も取ることができるでしょう。