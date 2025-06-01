---
title: ".d.tsファイルをちゃんと使うために必要な知識"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

**.d.tsファイル**とは、TypeScriptにおいて**型定義ファイル**と呼ばれるファイルのことです。残念なことに、.d.tsファイルは誤った使い方をされているのを見かけることがあります。そこで、この記事では、.d.tsファイルを正しく使うために必要な知識を解説します。

## .d.tsファイルとは

**.d.tsファイル**については、とりあえずTypeScript公式による以下の説明を読んでください（[Announcing TypeScript 5.5](https://devblogs.microsoft.com/typescript/announcing-typescript-5-5/#:~:text=Declaration%20files%20%28a.k.a.%20,declaration)から引用）。

> Declaration files (a.k.a. .d.ts files) describe the shape of existing libraries and modules to TypeScript. This lightweight description includes the library’s type signatures and excludes implementation details such as the function bodies. They are published so that TypeScript can efficiently check your usage of a library without needing to analyse the library itself. Whilst it is possible to handwrite declaration files, if you are authoring typed code, it’s much safer and simpler to let TypeScript generate them automatically from source files using --declaration.

**日本語訳**: Declaration files（.d.tsファイル）は、TypeScriptに対して既存のライブラリやモジュールの形状を説明するものです。これは軽量なもので、ライブラリの型シグネチャが含まれる一方、関数の本体などの実装の詳細は含まれません。.d.tsファイルを公開することで、TypeScriptがライブラリ自体を分析することなく、ライブラリの使用を効率的にチェックできるようになります。.d.tsファイルを手で書くことも可能ですが、あなたが書いているコードに型がある（訳注: JavaScriptではなくTypeScriptを書いている）場合は、`--declaration`オプションを使ってTypeScriptコンパイラに.d.tsファイルを自動生成させる方が、より安全かつ簡単です。

余談ですが、こういう公式の説明がどこに書いてあったか調べるのに各種AIのDeep Researchが便利ですね。特にTypeScriptはAnnouncing TypeScriptのブログ記事にさらっと重要な見解が書いてあったり、GitHubを見に行かないと重要な説明が無かったりします。

## .d.tsファイルはいつ使われるか

.d.tsファイルが現在最も使われる場面は、上記の説明にもあるように、**TypeScriptで書かれたコードをJavaScriptにコンパイルして、npmなどで配布するとき**です。例えば、以下の構成のプロジェクトを考えます。

```
src
├── index.ts
└── utils.ts
```

これをTypeScriptでコンパイルし、.d.tsファイルも出力した場合、出力は以下のようになります。

```
dist
├── index.js
├── index.d.ts
├── utils.js
└── utils.d.ts
```

そして、このdistディレクトリの中身をnpmにパッケージとして公開します。そうすれば、ライブラリのユーザーは.d.tsファイルを通じてライブラリの型情報を得ることができます。

この場合は、.d.tsファイルはコンパイラの成果物であり、手で書いたものではありません。これが、公式の説明にある「あなたが書いているコードに型がある場合は、`--declaration`オプションを使ってTypeScriptコンパイラに.d.tsファイルを自動生成させる方が、より安全かつ簡単です」ということです。元となるソースコードがTypeScriptの場合は、手で`.d.ts`ファイルを書く必要はありません。

言い換えると、筆者が冒頭で述べた誤った使い方とは、TypeScriptのプロジェクトなのに手で.d.tsファイルを書いているような状況を指します。

## .d.tsファイルのよくない使い方

筆者の見解では、.d.tsファイルは「型定義ファイル」とも呼ばれることから、**型定義を書くなら.d.tsファイルだという誤解**があるのではないかと思います。

```ts
// ユーザーの型定義だから src/types/user.d.ts に書こう！（誤解）
export interface User {
  id: string;
  name: string;
  email: string;
  // ...
}
```

この場合は、わざわざ.d.tsファイルを作る必要はありません。TypeScriptのプロジェクトであれば、`.ts`ファイルに型定義を書いてしまえばよいのです。今回の場合、（ディレクトリ構成の良し悪しはさておき）`src/types/user.ts`でいいのです。

## .d.tsファイルが存在することの意味

例えば、`src/foo.d.ts`というファイルがプロジェクト内にある場合、TypeScriptコンパイラは、これを「`src/foo.js`に対する型定義」であると認識します。もともと、.d.tsファイルというのは、同名の.jsファイルに対する型定義という意味なのです。前述のdistディレクトリの例を見ても、確かに.jsファイルと.d.tsファイルがセットになっていますね。

```
dist
├── index.js
├── index.d.ts
├── utils.js
└── utils.d.ts
```

逆に言えば、**対応する.jsファイルが無いのに.d.tsファイルだけあるのはおかしい**ということが、基本的な考え方として言えます。

例えば、先ほどの例のように`src/types/user.d.ts`というファイルがある場合、User型を使う側はこのようにimportすることになります。

```ts
import type { User } from './types/user';

const user: User = {
  id: "123",
  name: "John Doe",
  email: "john.doe@example.com"
};
```

実は、インポート元を次のようにすることもできます。そう、`user.d.ts`があることによって、TypeScriptはそのファイルが`user.js`に対する型定義であると認識しているのです。先ほどの`./types/user` も、.jsという拡張子が省略されているという扱いです。

```ts
import type { User } from './types/user.js';
```

特に、Node.jsのESM環境（`module: "nodenext"`）ではimport宣言での拡張子の省略ができないため、.jsと明示的に書く必要があります。

いずれにせよ、このような「本来存在しないファイルに対するimport」がうっかりランタイムに残ってしまったら、ランタイムエラーの原因になることも考えられます。

:::details 余談
実は、.d.tsファイルを`import type`でインポートすることがTypeScript 5.0以降で可能になりました。

```ts
import type { User } from './types/user.d.ts';
```

これは、これまでの「.d.tsファイルは.jsファイルに対する型定義」という説明に相反するものです。この書き方では、.d.tsファイルそのものの存在を明示的に取り扱っていることになりますね。

そのため筆者としては正直、これはできないほうがいいと思ったため、なぜこれが許されているのか調べました。理由は以下のコメントで説明されていました。

https://github.com/microsoft/TypeScript/pull/51669#issuecomment-1341805339

これによると、Project referencesなどで複数のプロジェクトにまたがるようなユースケースでの利便性の問題が念頭にあるようです。
:::

## プロジェクト全体で使う型定義

「プロジェクト全体で使う型定義」のようなものがtypes.d.tsみたいなファイルに書かれていることがたまにあります。そもそもディレクトリ設計をもっと良くして適切な場所に型定義を置くべきですが、それは脇に置いておきます。

このような場合も、.d.tsファイルを使う必要はありません。

ご存じの方が多いかもしれませんが、このような`src/types.d.ts`があった場合、exportとかせずに型を定義しておくだけで、全てのファイルでその型が使用できるようになりますね。

```ts
// src/types.d.ts
interface User {
  id: string;
  name: string;
  email: string;
  // ...
}
```

```ts
// src/index.ts

// importしていないのにUser型が使える！
const user: User = {
  id: "123",
  name: "John Doe",
  email: "john.doe@example.com"
};
```

これが.d.tsファイルの効果だと勘違いされているケースがたまにありますが、実は関係ありません。`src/types.d.ts`ではなく`src/types.ts`にしても、同じように全てのファイルで型が使えるようになります。

```ts
// src/types.ts
interface User {
  id: string;
  name: string;
  email: string;
  // ...
}
```

```ts
// src/index.ts
// importしていないのにUser型が使える！
const user: User = {
  id: "123",
  name: "John Doe",
  email: "john.doe@example.com"
};
```

この理由は、TypeScriptが.tsなどのソースファイルを**スクリプト**と**モジュール**に区分し、スクリプトのスコープはグローバルスコープであるとしているからです。この場合、`src/types.ts`はスクリプトとして扱われ、グローバルスコープに`User`型が定義されるため、他のファイルからも参照できるようになります。

### declare globalを使う

上述の話には注意点があります。それは、TypeScriptのオプションが`module: "nodenext"`のようなNode.js向けの設定になっている場合です。

この設定下では、package.jsonに`"type": "module"`が指定されていると、TypeScriptは全ての.tsファイルをモジュールとして扱います。つまり、`src/types.ts`もモジュールとなるので、グローバルスコープに型を定義することができません。

さらに、`"type": "module"`が指定されていない場合でも問題があります。`module: "nodenext"`の場合、`moduleDetection`というコンパイラオプションのデフォルト値が`force`になります。このオプションが設定されている場合、やはり全ての.tsファイルがモジュールとして扱われます。

このように、Node.js向けの設定では全てのファイルをモジュールとして扱うべく二重の仕組みが働いています。

そのような環境では、`declare global`構文を使うことで、やはり.tsファイルからグローバルスコープに型を定義することができます。

```ts
// src/types.ts
declare global {
  interface User {
    id: string;
    name: string;
    email: string;
    // ...
  }
}
```

```ts
// src/index.ts
// importしていないのにUser型が使える！
const user: User = {
  id: "123",
  name: "John Doe",
  email: "john.doe@example.com"
};
```

また、`declare global`は、types.tsがimport宣言を含むような場合でも使えます。TypeScriptは、import宣言を含むファイルをスクリプトとして扱いません。その場合はtypes.tsからグローバルスコープに型を定義するために、`declare global`を使う必要があります。

```ts
// src/types.ts
import type { UserId } from './id';

declare global {
  interface User {
    id: UserId;
    name: string;
    email: string;
    // ...
  }
}
```

### .d.tsファイルとモジュール

実は.d.tsファイルは、package.jsonに`"type": "module"`が書いてあったり、`moduleDetection`オプションが`force`に設定されていたりする場合でも、従来どおりの条件（ファイル中にimportやexportを含まない）を満たせばスクリプトとして扱われます。src/types.tsではグローバルな型を定義できなかったケースでも、内容は同じでファイル名をsrc/types.d.tsにすればグローバルな型が定義できるのです。

そのため、このように.d.tsを「モジュールではなくスクリプトである」ことを表すシグナルとして使う場合には、d.tsが便利なのは事実です。

ただ、この場合は上述のdeclare globalを使えばいいので、.d.tsを使わない選択肢もあります。

## アンビエントモジュール宣言の場合

例えばバンドラの設定で画像ファイルをimportできるようにしている場合は、以下のようなモジュール宣言（アンビエントモジュール宣言）が必要になります。

```ts
declare module '*.png' {
  const value: string;
  export default value;
}
```

このような宣言を.d.tsファイルに書いているケースがあります。しかし、これも実はグローバルな型定義の場合と同様に、スクリプト扱いのファイルであれば、.tsファイルに書いても問題ありません。

```ts
// src/pngs.ts
declare module '*.png' {
  const value: string;
  export default value;
}
```

```ts
// src/index.ts
// pngファイルをimportできる！
import logo from './logo.png';
```

そのため、基本的にはこれも.d.tsファイルを使う必要はないのですが、ひとつ例外があります。それは、前述の`moduleDetection: "force"`の場合です。この場合、.tsファイルは全てモジュール扱いになるので、`declare module '*.png'`を.tsファイルに書くとエラーが発生してしまいます。

:::details エラー内容

この場合に発生するエラーは以下のようなものです。

```
Invalid module name in augmentation, module '*.png' cannot be found.
```

これがどういうことなのか、詳しく知りたい方はこちらの記事がおすすめです。

https://zenn.dev/qnighy/articles/9c4ce0f1b68350

:::

実は、アンビエントモジュール宣言は`declare global`の中に書くことができません。そのため、この場合には、.d.tsファイルを用意して書く必要があります。

```ts
// src/pngs.d.ts
declare module '*.png' {
  const value: string;
  export default value;
}
```

## 余談: .d.ext.ts ファイルについて

実は、.d.tsファイルの陰に隠れて、.d.ext.tsというファイルもあります（ext部分には任意の拡張子が入ります）。これは、特定の拡張子を持つファイルに対する型定義を提供するためのものです。

例えば、特定の`.css`ファイルに対する型定義を提供するために、`styles.d.css.ts`というファイルを作成することができます。

```css
/* styles.css */
.foo {
  color: red;
}

.bar {
  color: blue;
}
```

```ts
// styles.d.css.ts
const styles: {
  foo: string;
  bar: string;
};

export default styles;
```

使う側は以下のようにインポートできます。

```ts
import styles from './styles.css';

console.log(styles.foo);
console.log(styles.bar);
```

ただし、このように.jsや.ts以外のファイルをimportするためには、TypeScriptの`allowArbitraryExtensions`オプションを有効にする必要があります。

### 従来の方法との比較

従来`styles.css`に型定義を与えてimportするためには、`styles.css.d.ts`というファイルを作成する方法がありました（拡張子の順番の違いに注意してください）。

```ts
// styles.css.d.ts
declare const styles: {
  foo: string;
  bar: string;
};
export default styles;
```

```ts
// 読み込める！
import styles from './styles.css';
```

ただ、この方法はある種のごまかしがありました。実は、この方法では、TypeScriptは`styles.css.js`というファイルを読み込んでいるつもりでいるのです。.jsという拡張子が省略されているわけですね。

「.d.tsファイルは同じ名前の.jsファイルに対する型定義である」という原則を思い出してください。この原則に従い、`styles.css.d.ts`は`styles.css.js`に対する型定義であるとTypeScriptは認識しているのです。

それに対して、`styles.d.css.ts`は、`styles.css`に対する型定義であることを明示的に表しています。.jsが省略されている扱いではありません。

従来の.d.tsではどうしても.jsファイルに対する型定義しか表せませんでした。それを補うのが.d.ext.tsファイルだというわけです。そう考えると、.d.ext.ts機能の追加と同時に`allowArbitraryExtensions`オプションが追加されたことも納得できますね。

特に、Node.jsのESM環境では拡張子を省略できないため、`allowArbitraryExtensions`オプションを有効にしていないと、`styles.css`をimportすることができません。

## まとめ

この記事では、.d.tsファイルの正しい使い方と、よくある誤解について解説しました。

.d.tsファイルは既存のJSファイルに対する型定義を表すものというのが原義であるため、そうではない用途で使うのは基本的には避けるべきです。

ただし、.d.tsファイルは.tsファイルでは代替できない独特の便利な挙動を備えています。特にNode.jsのESM環境や`moduleDetection: "force"`の環境では、この挙動に頼る場合があるかもしれません。

- グローバルスコープに型を定義したい場合（`declare global`でも代替可）
- アンビエントモジュール宣言を行う場合（`declare global`では代替不可）

### 参考リンク

この記事の内容は以下の記事と重複するところがあります。ただ、この記事が公開されて以降も誤解を見かけることがあるため、自分も記事を書いておこうと思いました。

https://zenn.dev/qnighy/articles/9a6a0041f2a1aa
