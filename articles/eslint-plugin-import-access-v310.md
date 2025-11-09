---
title: "eslint-plugin-import-accessのpackageDirectoryオプション"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

皆さんこんにちは。今回は、筆者が公開しているESlintプラグイン[eslint-plugin-import-access](https://github.com/uhyo/eslint-plugin-import-access)の新バージョン (v3.1.0) で追加された新しいオプション `packageDirectory` を紹介します。

このオプションにより、プラグインの活用の幅がかなり広がります。ぜひ試してみてください。

## 復習: eslint-plugin-import-accessとは

まず、このプラグイン自体をご存じない方のために、基本的なところを解説します。次の画像はeslint-plugin-import-accessの機能を端的に説明するものです。

![eslint-plugin-import-accessの機能を画像（このあと本文で説明されます）](/images/eslint-plugin-import-access-v310/concept.png)

eslint-plugin-import-accessが無い状態では、プロジェクト内のどこからどこへでも自由に`import`することができます。しかし、実際にはインポートに制限をかけたいことが多くあります。例えば、いわゆるpackage by featureでディレクトリを構成している場合、あるfeatureの中から、他のfeatureの内部の関数を自由に呼び出したりするのは避けたいです。

このような場合にeslint-plugin-import-accessを使うことで、インポートに制限をかけることができます。

上の画像では、`sub/foo.ts`から`const foo`がエクスポートされています。この`foo`を`sub/bar.ts`からインポートするのはOKですが、`sub`の外にある`main.ts`からインポートするのはエラーとなります。

つまり、この例では`sub`ディレクトリを「パッケージ」に見立てており、パッケージの中でのインポートは自由だが、パッケージの外（`main.ts`）からパッケージの中をインポートするのはだめということです。

このように、「パッケージ」の概念を採り入れることで自由なインポートを制限するのがeslint-plugin-import-accessの基本的な機能です。


### 他のimport系ルールとの違い

自由なインポートを制限するESLintルールは他にもありますが、それらとeslint-plugin-import-accessの大きな違いは**JSDocを用いた制御**にあります。

上の画像の例では、よく見ると以下のようにJSDocで`@package`が指定されていますね。

```ts
/**
 * @package
 */
export const foo = 3;
```

このように`@package`を指定することで、この`export`は「package-private」だよという意味になります。package-privateというのは「パッケージ外からインポートしてはいけない」という意味です。

つまり、eslint-plugin-import-accessでは、設定ファイルでルールを全部指定するのでなく、JSDocを用いることで一つ一つの`export`単位で「これはパッケージ外からインポートしてもいいのかどうか」を指定できるのです。

デフォルトでは、`@package`を指定したものだけがeslint-plugin-import-accessによる制限の対象となります。`@package`を指定していないものに関しては、外から自由にインポートできます。

しかし、`defaultImportability: "package"`オプションを指定することでデフォルトを`@package`にできます。この場合、JSDocでいちいち指示しなくても、全ての`export`が全部パッケージ外からインポート不可になります。このとき例外的にパッケージ外からインポート可能にするためには、`@public`をJSDocで指定します。

eslint-plugin-import-accessでプロジェクト全体を統制したい場合に`defaultImportability: "package"`が活用されています。

## 新オプション `packageDirectory`

これまで「パッケージ」「パッケージ外」といった説明をしてきましたが、パッケージとは何を指すのでしょうか。

実は、これまでのeslint-plugin-import-accessでは**パッケージ＝ディレクトリ**であり、全てのディレクトリがeslint-plugin-import-accessからはパッケージとして扱われていました。

つまり、`@package`扱いになっているファイルは、そのファイルがあるディレクトリの外からインポート不可ということです。

これはシンプルなルールではありますが、あまり融通がきかなくて不便でした。

そこで今回登場したオプションが`packageDirectory`です。これはglobパターンを使って、**どのディレクトリをパッケージとして扱うか指定できる**というものです。以下のような指定ができます。

```ts
// デフォルト（全てのディレクトリがパッケージ）
packageDirecotry: ["**"] 
// _internalという名前のディレクトリはパッケージとして扱わない
packageDirectory: ["**", "!**/_internal"]
// src/packages直下のディレクトリがパッケージとなる
packageDirectory: ["src/packages/*"]
```

例えば、`_internal`というディレクトリ名をパッケージとして扱わないようにしたとします（このユースケースは[@honey32](https://github.com/uhyo/eslint-plugin-import-access/issues/126)さんに提供いただきました）。

この場合、以下のような挙動になります。

```ts
// ----- src/feature1/_internal/helpers.ts
/**
 * @package
 */
export const internalHelper = () => { /* ... */ };

// ----- src/feature1/foo.ts
import { internalHelper } from "./_internal/helpers"; // ✓ インポート可能

// ----- src/main.ts
import { internalHelper } from "./feature1/_internal/helpers"; // ✖ インポート不可
```

まず、`internalHelper`は`@package`でエクスポートされていますが、これが属するパッケージは`src/feature1`になります。直接の親ディレクトリは`src/feature1/_internal`ですが、これはパッケージとして扱わないため、さらにその親が所属パッケージになります。

そうなると、`src/feature1/foo.ts`から`internalHelper`をインポートすることが可能になります。このファイルも`src/feature1`に属しており、同じパッケージ内でのインポートになるからです。

一方、`src/main.ts`からインポートすることはできません。なぜなら、`main.ts`は`src/feature1`パッケージの外にあり、外から中をインポートすることは禁止だからです。

このように、`packageDirectory`があると、パッケージ内でのファイルの整理の自由度が上がります。

## おすすめの使用法

筆者は、`packageDirectory`の登場により、上で言及した`defaultImportability: "package"`の実用性が向上したと考えています。従来は厳しすぎて使うのに苦労するオプションでしたが、どの単位を「パッケージ」として扱うのか設定できるようになったことで、過度な制限をかけることなくプロジェクトの秩序を保つことができます。

`packageDirectory`を用いて適当な大きさにパッケージを切って、`defaultImportability: "package"`を用いてパッケージ間のインポートを制限しましょう。

そして、特別に他のパッケージへの提供を意図しているexportにのみ`@public`をつけましょう。

こうすることで、そのパッケージのインターフェース（パッケージ外に対してどのような機能を提供しているのか）が明確になります。

例えば、フロントエンドのアプリケーションで画面単位のpackageとした場合、画面全体のコンポーネントだけ`@public`にする運用が考えられます。

```tsx
// src/features/foo

/**
 * @public
 */
export const FooPage: React.FC = ...
```

## “メタ設定”のすすめ

このように便利な`packageDirectory`ですが、設定の書き方にはコツがあります。それは、**個別のディレクトリを列挙するのではなく、“ディレクトリ名のルール”を`packageDirectory`に記載すること**です。より端的に言えば、ワイルドカード（`*`や`**`）を活用するということです。

例えば、いわゆるpackage by featureを採用しており`src/packages/foo`や`src/packages/bar`をパッケージとして扱いたいとしましょう。そのときは、`foo`や`bar`を個別に設定ファイルに明記するのは避け、代わりに`*`を使います。

```ts
// 避けるべき設定方法
packageDirectory: ["src/packages/foo", "src/packages/bar"],
// 望ましい設定方法
packageDirectory: ["src/packages/*"],
```

前者のようなやり方は、パッケージが増えるたびにESLintの設定ファイルを書き換える必要がある点が特によくありません。より抽象度の高いルール設定にすることで、このような事態を避けられます。

他に、以下のような設定にしてみるのも面白いでしょう。

```ts
// foo.package のような名前のディレクトリはパッケージとして扱う
packageDirectory: ["**/*.package"],
// bar.internal のような名前のディレクトリはパッケージとして扱わない
packageDirectory: ["**", "!**/*.internal"],
```

上記のような設定の場合は、一度ルールを定義してしまえば、あとは個々のディレクトリがパッケージかどうかは、ディレクトリの命名で決めることができます。つまり、ディレクトリを作る人が、そのディレクトリに関するルールも決めることができます。

このやり方では、ESLint設定 (`packageDirectory`) は言わば「ルールを決めるルール」になっていますね。これを筆者は**メタ設定**と呼んでいます。Next.jsのフレームワークにファイル・ディレクトリの命名ルールがあるのと同じようなものですね。eslint-plugin-import-accessを使うことで自分で命名ルールを定義できるのです。

## まとめ

eslint-plugin-import-accessに新しいオプション`packageDirectory`が追加されたことで、eslint-plugin-import-accessの利便性と活用の幅が広がりました。

特に、このオプションがあることで`defaultImportability: "package"`での運用が現実的なものになります。自分のプロジェクトにの状況や構成にあわせて`packageDirectory`のルールを定義しましょう。これにより、望ましくないインポートを効率よく検知できるようになります。

また、“メタ設定”の考え方は、eslint-plugin-import-accessがもともと持っていた「JSDocを用いて、設定ファイルではなく現場でルールを指定する」という特徴をさらに推し進めるものとなっています。JSDocに加えて、ディレクトリ名という道具が新たに加わったわけですね。

ちなみに、今回のオプションの実装は、Claude Code on the webのクレジットが何か付与されていたので使ってみる目的で全てClaude Codeにやってもらいました。多分大丈夫だと思いますが、使ってみて問題があればissueなどでの共有をぜひよろしくお願いいたします。