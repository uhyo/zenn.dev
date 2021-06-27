---
title: "eslint-plugin-import-accessではじめるディレクトリ単位カプセル化"
emoji: "📂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "eslint"]
published: true
---

こんにちは。この記事は筆者が製作したESLint向けプラグイン [eslint-plugin-import-access](https://github.com/uhyo/eslint-plugin-import-access)を紹介する記事です。

このプラグインによりTypeScriptプログラムに擬似的な**package-private export**の概念が生まれます。JSDocで`@package`とアノテートされた`export`宣言は、そのファイルが属するディレクトリの外からインポートすることができなくなります。

従来、TypeScriptで可能なカプセル化の最大の単位は「ファイル」であり、ファイルからエクスポートしない変数はそのファイル（モジュール）の中に閉じている一方で、一旦エクスポートしたものはプロジェクトのどこからでもインポート可能になります。これでは不都合な場合がありました。

最近の具体的な例としては[Recoil](https://recoiljs.org/)が挙げられます。[筆者の以前の記事](https://blog.uhy.ooo/entry/2020-05-16/recoil-first-impression/)では、AtomやSelectorといったものをエクスポートするのではなくそれらをラップしたカスタムフックをエクスポートするのが良いと述べました。しかし、Selectorのネットワークを張り巡らせようとすると、Selectorをエクスポートできないためひとつのファイルが肥大化してしまいます。

このような問題を解決するために、`eslint-plugin-import-access`は「ディレクトリ」というより大きな単位のカプセル化を提供します。

これは[TypeScript本体の機能として欲しいというissue](https://github.com/microsoft/TypeScript/issues/41425)を上げており70以上の👍を得ていましたがTypeScriptチームに動きはなく、仕方ないのでESLintプラグインとして作ったものです。

## 具体例

次のように `src/sub/foo.ts` からエクスポートされた変数たちを考えてみましょう。普通の`export`はpublic（どこからもインポートできる）な従来通りの`export`です。`@package`はpackage-privateになります。また、おまけとして`@private`も実装しました。これはどこからもインポートできないという意味です[^note_1]。

[^note_1]: それなら`export`する意味がないように思えますが、一応ignore commentを書いた場合だけインポートできるので何か使い道があるかもしれません。

```ts:src/sub/foo.ts
export const subPublic = "Hi, I'm public"

/**
 * @package
 */
export const subPackage = "package private"

/**
 * @private
 */
export const subPrivate = "HIDDEN"
```

これらを同じ`sub`ディレクトリ内の`index.ts`からインポートする場合、publicエクスポートもpackageエクスポートもインポート可能です。privateエクスポートはESLintによりエラーが発生します。

```ts:src/sub/index.ts
import { subPublic, subPackage, subPrivate } from "./foo";
//                              ^^^^^^^^^^ ここにエラー発生
console.log(subPublic, subPackage, subPrivate);
```

![VSCodeのスクリーンショット](https://storage.googleapis.com/zenn-user-upload/1a2d790dd77d0a4c230308d4.png)

一方で、`sub`ディレクトリの外のファイルからインポートする場合、`sub`の中のpackageエクスポートをインポートすることはできなくなります。

```ts:src/
import { subPublic, subPackage, subPrivate } from "./sub/foo";
//                  ^^^^^^^^^^  ^^^^^^^^^^ ここにエラー発生
console.log(subPublic, subPackage, subPrivate);
```

![VSCodeのスクリーンショット](https://storage.googleapis.com/zenn-user-upload/b12063c2df742be252b3e13f.png)

このように、ファイルの位置関係によってpackageエクスポートをインポートできるか否かが決まります。

## TypeScriptのディレクトリ構成が意味を持つ

これまで、TypeScriptプロジェクトのディレクトリ構成というのは、（フレームワークによって決められたものを除けば）我々の気分で好きに決めていました。従来はどこで`export`された変数をどこから`import`することもできるのであり、ディレクトリ構成の意味はプロジェクトの見通しをよくするといったあくまで管理上のもので、プログラムそのものとは無関係でした。

この記事の冒頭でも述べたように、`eslint-plugin-import-access`によってディレクトリ構成に「カプセル化」という実利上の意味が与えられることになります。これにより、今後のTypeScriptプロジェクトでは`@package`の利用を念頭に置いたディレクトリ構成が考えられることになります。とても楽しみですね。

## TypeScript Language Service Plugin

勘のいい読者の方は、ESLintだけでは開発体験が十分でないことにお気づきでしょう。良いエディタを使ってTypeScriptを書いている方にとっては、`import`文は自分で書くものではなく自動的に補完されるものです。

一方、ESLintはあくまで記述されたコードの問題点を検出するものですから、勝手にpackage-privateなエクスポートが`import`されて怒られることが想定されます。これは開発体験がよくありません。

そこで、`eslint-plugin-import-access`にはこの点を補うTypeScript Server Pluginが同梱されています。これを有効にすることで、`@package`または`@private`が付いているためインポートできないものは補完から除外されます。

VSCodeを使っている場合の動作例を示します。上の例の`src/index.ts`で`subP`まで入力すると、`subPublic`のみが自動インポート候補として表示されます。

![src/index.tsのスクリーンショット](https://storage.googleapis.com/zenn-user-upload/48f29455ad1ffbbd153fc120.png)

一方、`src/sub/index.ts`で同様に`subP`まで入力すると、`subPublic`と`subPackage`が候補に表示されます。この位置では同じディレクトリ内の`src/sub/foo.ts`からエクスポートされた`subPackage`をインポートすることができるので、それが補完候補にもきちんと現れています。

![src/sub/index.tsのスクリーンショット](https://storage.googleapis.com/zenn-user-upload/b9ad8a66b0789daad50d5d67.png)

このように、TypeScript Language Service Pluginを使うことで開発体験を損なわずに`eslint-plugin-import-access`の恩恵を受けることができます。

## 2種類の抜け穴

実は、`eslint-plugin-import-access`には2種類の抜け穴 (loophole) が用意されています。これらはオプションで有効にするかどうかを決めることができ、デフォルトでは`index.ts`のほうのみが有効になっています。

### `index.ts`による抜け穴

TypeScriptでは、`import { ... } from "./sub"`のようにすることで`sub/index.ts`からインポートすることができます[^note_2]。これに対応して`index.ts`を特別扱いするという抜け穴がデフォルトで有効になっています。

[^note_2]: 正確には `moduleResolution`コンパイラオプションを`"node"`にしている場合。このことから分かるように、TypeScript特有ではなくNode.jsの挙動に由来するものです（CommonJSの場合）。

例えば、`sub/index.ts`で`@package`エクスポートされたものは、`sub`ディレクトリの隣にあるファイルからも読み込むことができるようになります。これは、`sub`内のpackage-privateな機能たちを**再エクスポート**したいときに便利です。次の例では、`sub/index.ts`は`foo`を`@package`で再エクスポートします。こうすると、`sub`の親ディレクトリからも`foo`が利用できるようになります。

```ts:src/sub/foo.ts
/**
 * @package
 */ 
export const foo = "FOO";
/**
 * @package
 */
export const bar = "BAR";
```

```ts:src/sub/index.ts
import { foo } from "./foo";

/**
 * fooを再エクスポート
 * @package
 */
export { foo };
```

```ts:src/user.ts
// 読み込める！
import { foo } from "./sub";
```

これにより、ディレクトリ単位の“パッケージ”がネストするような場合には、`index.ts`をサブパッケージの窓口とする運用が可能です。

### ファイル名による抜け穴

以上のような抜け穴は再エクスポートを可能にするために用意されたものですが、`index.ts`にそのような特別な役割を持たせるべきではないという意見がもしかしたらあるかもしれません。

というのも、Node.jsでもECMAScript Modulesがサポートされていますが、こちらには`index.js`を特別扱いするような機構はないのです。`./sub`ではなく`./sub/index.js`としないと`index.js`を読み込めません。

そのような状況で、`index.ts`に新たな特別性を持たせるのは望ましくないという意見があるかもしれません。そこで、もう一つ用意された抜け穴がファイル名による抜け穴です。こちらはデフォルトでは有効になっていないので、使用する場合はオプションで有効にする必要があります。

こちらの抜け穴は、**ディレクトリ名と同じファイルからはディレクトリの中のpackageエクスポートをインポートできる**というものです。例えば、`sub.ts`は`sub/`直下のファイルのpackageエクスポートをインポートできます。つまり、`sub/index.ts`ではなく`sub`ディレクトリと同じ位置に並べた`sub.ts`を`sub/`以下の窓口にしようという考え方です。

```ts:src/sub/foo.ts
/**
 * @package
 */ 
export const foo = "FOO";
```

```ts:src/sub.ts
// 読み込める！
import { foo } from "./sub/foo";
```

このようにディレクトリと同名のファイルを窓口にするのがどれくらいメジャーなのかは寡聞にして分かりませんが、Rust（2018エディション）で採用されているのを目にしたので取り入れました。

## まとめ

この記事では、TypeScriptプロジェクトにディレクトリレベルのpackage-privateエクスポートという新しい概念をもたらすために作られた、ESLintルールおよびTypeScript Language Service Pluginである`eslint-plugin-import-access`を宣伝しました。

https://github.com/uhyo/eslint-plugin-import-access

現在は`1.0.0-beta`が公開されています。すこしドッグフーディングしてみて正式リリースに持っていきたいなと思っています。いいなと思った方はぜひ使ってみてください。リポジトリへの ⭐️ もぜひお願いします。