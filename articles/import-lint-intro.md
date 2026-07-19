---
title: "ImportLintの紹介: import専門の高速なリンター"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "importlint"]
published: true
---

皆さんこんにちは。今回は、筆者が最近Claude Fable 5さんと一緒に開発した**ImportLint**を紹介します。

https://github.com/uhyo/import-lint

## eslint-plugin-import-accessを知っている・使っている方向けの概要

ImportLintは、筆者がESLintプラグインとして公開していた[eslint-plugin-import-access](https://github.com/uhyo/eslint-plugin-import-access)を移植し、Rust製の独立したCLIツールとして再実装したものです。eslint-plugin-import-accessの全てのオプションをサポートしており、簡単に移行できます。

開発した動機は、eslint-plugin-import-accessがESLintプラグインで、しかもTypeScriptの型情報に依存していたため、oxlintのような他のリンターへの移行を妨げてしまうことでした。

このESLintプラグインはTypeScriptの型情報に依存していましたが、実はそれはimportの解決のためだけでした。

そこで、JS/TSのパースをoxcで行い、importの解決もoxc-resolverに頼ることでESLintに依存せず、しかも高速化もできそうだというアイデアがありました。実際に、このツールはESLint版に比べて非常に高速で、このルールのためだけにESLintを動かした場合と比較すると、100倍以上高速（150万行規模のプロジェクトでも開発機では1～2秒）という結果が出ています。

ESLintの速度にお困りの方は、ぜひこちらを試してみてください。

## ImportLintとは

では、eslint-plugin-import-accessを知らない方も含めて理解いただけるように、ImportLintの概要を説明していきます。

このツールでできることは、**パッケージをまたいだimportを制限すること**です（ただし、パッケージというのは `package.json` とかnpmのパッケージのことではなく、プロジェクト内の区分けのことです。これはあとで説明します）。

TypeScriptプロジェクトでは、せっかくコードベースをきれいに整理してディレクトリ分けしても、好きなファイルから好きなファイルをimportできてしまいます。例えば、いわゆるpackage by featureを実践している場合、featureまたぎのimportが発生してしまったりすると、コードの保守性が低下する要因になります。

ImportLintは、**importの制限ルールを設定し、違反しているimportを検出する**ツールです。これにより、importに秩序をもたらし、コードの保守性を高めることができます。

しかし、そのようなツールは、eslint-plugin-import系列とか、いくらでもありそうに思いますよね。それらと比較したImportLintの顕著な特徴は、**パッケージの概念**をコードベースに導入することです。

## パッケージによる制御

ImportLintが現在 `import-lint init` で生成する設定ファイルには、推奨設定としてこんな設定が含まれています。

```json
{
  "defaultImportability": "package",
  "packageDirectory": ["**/*.package"]
}
```

特に注目すべきは`packageDirectory`で、この設定の意味は「`*.package`という名前のディレクトリをパッケージとして扱う」ということです。つまり、コードベースにおいて、`*.package`ディレクトリがパッケージの境界となります。

そして、ImportLintがやってくれることは、**パッケージの中のファイルを、パッケージの外のファイルからimportできないように制御する**ことです。

**例:** こんなimport関係を考えてみましょう。

```ts:src/billing.package/invoice.ts
export function computeInvoice(cents: number): number {
  return cents;
}
```

```ts:src/report.ts
import { computeInvoice } from "./billing.package/invoice";

console.log(computeInvoice(100));
```

`invoice.ts`は`billing.package`というパッケージの中にありますが、`report.ts`はパッケージの外にあります。つまり、これはパッケージの外からパッケージの中のファイルをimportしている状態です。ImportLintはこのimportを検出し、警告を出します。

```
src/report.ts
  1:10  error  Cannot import a package-private export 'computeInvoice'  package-access

✖ 1 problem (1 error, 0 warnings)
```

この制御により、パッケージ外で使われていることを想定していない関数や変数が、パッケージ外からimportされることを防ぐことができます。

ちなみに、設定にある`"defaultImportability": "package"`がこの動作に関係しています。この設定は、exportをデフォルトでパッケージ内でのみimport可能にするという意味です。

## lintエラーの修正方法

ImportLintの警告を修正するには、パッケージの外から中のものをimportしないようにすればいいわけです。しかし、場合によっては、パッケージの外からimportされることを意図したexportもあるでしょう。その場合の修正方法が2パターンあります。

### 1. `index.ts`からエクスポートする

ImportLintは、`index.ts`をパッケージの入口として見なします（`"indexLookhole": true`がデフォルトで有効になっています）。そのため、パッケージの外からimportされることを意図したexportは、`index.ts`からexportすることで、パッケージの外からimport可能になります。

```ts:src/billing.package/index.ts
// computeInvoiceからexport
export { computeInvoice } from "./invoice";
```

```ts:src/report.ts
// これはOK
import { computeInvoice } from "./billing.package";
// これはエラー
import { computeInvoice } from "./billing.package/invoice";
```

このように、`index.ts`を通じてパッケージのインターフェースを定義することで、パッケージの外に公開するexportを選択できます。

### 2. `@public`アノテーションを付ける

`index.ts`からは公開できないけど外からimportできるようにしたいという場合は、JSDocで`@public`アノテーションを付けることで、パッケージの外からimport可能にすることができます。

```ts:src/billing.package/invoice.ts
/** @public */
export function computeInvoice(cents: number): number {
  return cents;
}
```

ただ、これは常用するものではなく、エスケープハッチと考えたほうがよいでしょう。`@public`を付けるとプロジェクト内どこからでもimport可能になります。`index.ts`の場合は、あくまで`billing.package`のレベルで公開されることになり、もしパッケージがネストしていたらさらに外側まで脱出することはありません。

## `defaultImportability`の設定値

先ほどお見せした設定には`"defaultImportability": "package"`という設定がありました。これは、exportをデフォルトでパッケージ内でのみimport可能にするという意味です。

もしこれを`"defaultImportability": "public"`に変更した場合、デフォルトでは全部 `@public` 扱いになり、制限がなくなります。この場合、制限をかけたいexportに`@package`アノテーションを付けることで、パッケージ内でのみimport可能にすることができます。

```ts:src/billing.package/invoice.ts
/** @package */
export function computeInvoice(cents: number): number {
  return cents;
}
```

昔は`packageDirectory`が無く、`"defaultImportability": "package"`が厳しすぎて運用できなかったため`public`が主流でした。しかし、現在は`packageDirectory`を使うことで、パッケージの境界を明示的に定義できるようになったため、`package`を前提として運用するのが望ましいと考えています。

ちなみに、`packageDirectory`を設定しない場合のデフォルトは「全てのディレクトリがパッケージ」となり、`"defaultImportability": "package"`と組み合わせた場合全てのディレクトリにindex.tsが必要になるという結構壮絶な感じになっていました。

## メタ設定という考え方に基づく`packageDirectory`の設定

これはその`packageDirectory`オプションの[紹介記事](https://zenn.dev/uhyo/articles/eslint-plugin-import-access-v310)にも書いたことですが、筆者の好きな考え方として、**メタ設定**という考え方があります。

推奨の`"packageDirectory": ["**/*.package"]`という設定は、具体的にどのディレクトリがパッケージだ、ということを列挙するものではありません。そうではなく、**命名ルール**を定義しています。

これにより、実際にどのディレクトリをパッケージにするかという**具体的なルール**は、そのコードのオーナーが、ディレクトリの命名により決めることができます。いちいちImportLintの設定を変更する必要はありません。つまり、ImportLintの設定は「ルールを決めるための命名ルール」であり、言い換えるとルールのルールです。これが筆者がメタ設定と呼ぶ所以です。

このメタ設定という考え方に基づく限り、`packageDirectory`をどんな設定にしてもいいと筆者は考えています。もし`*.package`という命名ルールが気に入らなければ、別のルールにすればいいのです。

例えば、package-by-featureを意識している場合、こんな設定も考えられます。

```json
{
  "defaultImportability": "package",
  "packageDirectory": ["src/feature/*"]
}
```

これは、`src/feature`直下の各ディレクトリがパッケージになるという意味です。これなら、特別な命名規則なしに、feature間のimportを制限することができます。

もちろん、`packageDirectory`は配列ですから、複数のルールを組み合わせることもできます。例えば、`src/feature/*`と`src/lib/*`をパッケージとして扱いたい場合は、こんな設定になります。

```json
{
  "defaultImportability": "package",
  "packageDirectory": ["src/feature/*", "src/lib/*"]
}
```

さらに、`!`による除外をルールに組み込むこともできます。例えば、`_internal`という名前のディレクトリを除いて全てをパッケージとして扱いたい場合は、こんな設定になります。

```json
{
  "defaultImportability": "package",
  "packageDirectory": ["**", "!**/_internal"]
}
```

自分のプロジェクトにフィットするメタ設定を考えてみてください。

## 今さらImportLintの導入法・使い方

すごく順番が前後した気がしますが、ImportLintの導入法・使い方を簡単に紹介します。

ImportLintはnpmでインストールできます。

```bash
npm install -D @import-lint/cli
```

インストールしたら、`import-lint init`コマンドで設定ファイルを生成します。

```bash
npx import-lint init
```

`.importlintrc.jsonc`という設定ファイルが生成され推奨設定が書き込まれていますので、、必要に応じて設定を変更してください。

lintの実行も`import-lint`コマンドで行えます。

```bash
import-lint
```

推奨設定では、もともと`*.package`というディレクトリを作っていない限りはlintエラーが出ないはずです。まだパッケージ境界が切られていないので、プロジェクト全体が同じパッケージになり、自由に相互importができます。

試しに適当なディレクトリを`*.package`の形にリネームしてみることでエラーが検出できるでしょう。

設定ファイルの書き方などはGitHubを参照してください。

ちなみに、[VS Code拡張](https://marketplace.visualstudio.com/items?itemName=uhyo.import-lint)も用意してあります。AIにお願いしたら作ってくれるので良い時代ですね。また、`import-lint lsp`としてLSPサーバーも組み込まれていますので、他のエディタへの組み込みも可能なはずです。

また、今どきのツールらしく、AIにImportLintのことを理解してもらうための[スキル](https://github.com/uhyo/import-lint/blob/master/skills/import-lint/SKILL.md)も用意してあります。色々サボってGitHubからコピペしてもらう形になりました。

## 今後の展望

このツールは、eslint-plugin-import-accessの移植というところから始まりましたが、あえてImportLintという広めの名前をつけてみました。

今後、import関係のルールを拡充できたら面白いのではないかと考えています。

## まとめ

ディレクトリ構成は大規模なプロジェクトの保守性を維持するために重要な要素のひとつです。特に、適切にモジュールを切り、良いインターフェースを保つのが鍵になります。

ImportLintは、パッケージの概念をコードに導入し、importの秩序を保つためのツールです。

ぜひお好みのメタ設定を用意してプロジェクトに導入してみてください。感想をお待ちしています。