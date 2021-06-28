---
title: "eslint-plugin-import-accessではじめるディレクトリ単位カプセル化"
emoji: "📂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "eslint"]
published: true
---

こんにちは。この記事は筆者が製作した ESLint 向けプラグイン [eslint-plugin-import-access](https://github.com/uhyo/eslint-plugin-import-access)を紹介する記事です。

このプラグインにより TypeScript プログラムに擬似的な**package-private export**の概念が生まれます。JSDoc で`@package`とアノテートされた`export`宣言は、そのファイルが属するディレクトリの外からインポートすることができなくなります。

従来、TypeScript で可能なカプセル化の最大の単位は「ファイル」であり、ファイルからエクスポートしない変数はそのファイル（モジュール）の中に閉じている一方で、一旦エクスポートしたものはプロジェクトのどこからでもインポート可能になります。これでは不都合な場合がありました。

最近の具体的な例としては[Recoil](https://recoiljs.org/)が挙げられます。[筆者の以前の記事](https://blog.uhy.ooo/entry/2020-05-16/recoil-first-impression/)では、Atom や Selector といったものをエクスポートするのではなくそれらをラップしたカスタムフックをエクスポートするのが良いと述べました。しかし、Selector のネットワークを張り巡らせようとすると、Selector をエクスポートできないためひとつのファイルが肥大化してしまいます。

このような問題を解決するために、`eslint-plugin-import-access`は「ディレクトリ」というより大きな単位のカプセル化を提供します。

これは[TypeScript 本体の機能として欲しいという issue](https://github.com/microsoft/TypeScript/issues/41425)を上げており 70 以上の 👍 を得ていましたが TypeScript チームに動きはなく、仕方ないので ESLint プラグインとして作ったものです。

## 具体例

次のように `src/sub/foo.ts` からエクスポートされた変数たちを考えてみましょう。普通の`export`は public（どこからもインポートできる）な従来通りの`export`です。`@package`は package-private になります。また、おまけとして`@private`も実装しました。これはどこからもインポートできないという意味です[^note_1]。

[^note_1]: それなら`export`する意味がないように思えますが、一応 ignore comment を書いた場合だけインポートできるので何か使い道があるかもしれません。

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

これらを同じ`sub`ディレクトリ内の`index.ts`からインポートする場合、public エクスポートも package エクスポートもインポート可能です。private エクスポートは ESLint によりエラーが発生します。

```ts:src/sub/index.ts
import { subPublic, subPackage, subPrivate } from "./foo";
//                              ^^^^^^^^^^ ここにエラー発生
console.log(subPublic, subPackage, subPrivate);
```

![VSCodeのスクリーンショット](https://storage.googleapis.com/zenn-user-upload/1a2d790dd77d0a4c230308d4.png)

一方で、`sub`ディレクトリの外のファイルからインポートする場合、`sub`の中の package エクスポートをインポートすることはできなくなります。

```ts:src/
import { subPublic, subPackage, subPrivate } from "./sub/foo";
//                  ^^^^^^^^^^  ^^^^^^^^^^ ここにエラー発生
console.log(subPublic, subPackage, subPrivate);
```

![VSCodeのスクリーンショット](https://storage.googleapis.com/zenn-user-upload/b12063c2df742be252b3e13f.png)

このように、ファイルの位置関係によって package エクスポートをインポートできるか否かが決まります。

## TypeScript のディレクトリ構成が意味を持つ

これまで、TypeScript プロジェクトのディレクトリ構成というのは、（フレームワークによって決められたものを除けば）我々の気分で好きに決めていました。従来はどこで`export`された変数をどこから`import`することもできるのであり、ディレクトリ構成の意味はプロジェクトの見通しをよくするといったあくまで管理上のもので、プログラムそのものとは無関係でした。

この記事の冒頭でも述べたように、`eslint-plugin-import-access`によってディレクトリ構成に「カプセル化」という実利上の意味が与えられることになります。これにより、今後の TypeScript プロジェクトでは`@package`の利用を念頭に置いたディレクトリ構成が考えられることになります。とても楽しみですね。

## TypeScript Language Service Plugin

勘のいい読者の方は、ESLint だけでは開発体験が十分でないことにお気づきでしょう。良いエディタを使って TypeScript を書いている方にとっては、`import`文は自分で書くものではなく自動的に補完されるものです。

一方、ESLint はあくまで記述されたコードの問題点を検出するものですから、勝手に package-private なエクスポートが`import`されて怒られることが想定されます。これは開発体験がよくありません。

そこで、`eslint-plugin-import-access`にはこの点を補う TypeScript Server Plugin が同梱されています。これを有効にすることで、`@package`または`@private`が付いているためインポートできないものは補完から除外されます。

VSCode を使っている場合の動作例を示します。上の例の`src/index.ts`で`subP`まで入力すると、`subPublic`のみが自動インポート候補として表示されます。

![src/index.tsのスクリーンショット](https://storage.googleapis.com/zenn-user-upload/48f29455ad1ffbbd153fc120.png)

一方、`src/sub/index.ts`で同様に`subP`まで入力すると、`subPublic`と`subPackage`が候補に表示されます。この位置では同じディレクトリ内の`src/sub/foo.ts`からエクスポートされた`subPackage`をインポートすることができるので、それが補完候補にもきちんと現れています。

![src/sub/index.tsのスクリーンショット](https://storage.googleapis.com/zenn-user-upload/b9ad8a66b0789daad50d5d67.png)

このように、TypeScript Language Service Plugin を使うことで開発体験を損なわずに`eslint-plugin-import-access`の恩恵を受けることができます。

## 2 種類の抜け穴

実は、`eslint-plugin-import-access`には 2 種類の抜け穴 (loophole) が用意されています。これらはオプションで有効にするかどうかを決めることができ、デフォルトでは`index.ts`のほうのみが有効になっています。

### `index.ts`による抜け穴

TypeScript では、`import { ... } from "./sub"`のようにすることで`sub/index.ts`からインポートすることができます[^note_2]。これに対応して`index.ts`を特別扱いするという抜け穴がデフォルトで有効になっています。

[^note_2]: 正確には `moduleResolution`コンパイラオプションを`"node"`にしている場合。このことから分かるように、TypeScript 特有ではなく Node.js の挙動に由来するものです（CommonJS の場合）。

例えば、`sub/index.ts`で`@package`エクスポートされたものは、`sub`ディレクトリの隣にあるファイルからも読み込むことができるようになります。これは、`sub`内の package-private な機能たちを**再エクスポート**したいときに便利です。次の例では、`sub/index.ts`は`foo`を`@package`で再エクスポートします。こうすると、`sub`の親ディレクトリからも`foo`が利用できるようになります。

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

というのも、Node.js でも ECMAScript Modules がサポートされていますが、こちらには`index.js`を特別扱いするような機構はないのです。`./sub`ではなく`./sub/index.js`としないと`index.js`を読み込めません。

そのような状況で、`index.ts`に新たな特別性を持たせるのは望ましくないという意見があるかもしれません。そこで、もう一つ用意された抜け穴がファイル名による抜け穴です。こちらはデフォルトでは有効になっていないので、使用する場合はオプションで有効にする必要があります。

こちらの抜け穴は、**ディレクトリ名と同じファイルからはディレクトリの中の package エクスポートをインポートできる**というものです。例えば、`sub.ts`は`sub/`直下のファイルの package エクスポートをインポートできます。つまり、`sub/index.ts`ではなく`sub`ディレクトリと同じ位置に並べた`sub.ts`を`sub/`以下の窓口にしようという考え方です。

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

このようにディレクトリと同名のファイルを窓口にするのがどれくらいメジャーなのかは寡聞にして分かりませんが、Rust（2018 エディション）で採用されているのを目にしたので取り入れました。

## まとめ

この記事では、TypeScript プロジェクトにディレクトリレベルの package-private エクスポートという新しい概念をもたらすために作られた、ESLint ルールおよび TypeScript Language Service Plugin である`eslint-plugin-import-access`を宣伝しました。

https://github.com/uhyo/eslint-plugin-import-access

現在は`1.0.0-beta`が公開されています。すこしドッグフーディングしてみて正式リリースに持っていきたいなと思っています。いいなと思った方はぜひ使ってみてください。リポジトリへの ⭐️ もぜひお願いします。

### 補足

実は、ESLint で import の可不可を制御できるものは[eslint-plugin-import](https://github.com/benmosher/eslint-plugin-import)など既存のプラグインがあります。今回作った`eslint-plugin-import-access`がそれらと一線を画しているのは、`export`宣言に直接`@package`というアノテーションを書くことで`export`単位で制御できることです。既存のソリューションでは設定ファイルで制御することになり、プロジェクトの大ざっぱな区分けをすることはできますが小さなディレクトリ単位までカプセル化の恩恵を受けるのには適していません。`eslint-plugin-import-access`により、気軽に package-private export が使えるようになったのです。
