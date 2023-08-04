---
title: "TypeScriptのmoduleプションの話、あるいはTypeScript開発者の苦悩、あるいはCJSとESMの話"
emoji: "⚱️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

皆さんこんにちは。早速ですが、TypeScriptの`module`オプションはご存じでしょうか。`module`オプションは、例えば次のような値をサポートしています。

- `commonjs`
- `umd`
- `es2015`
- `esnext`
- `node16`
- `nodenext`

皆さんは、`module`オプションが何を設定するオプションなのか一言で説明できますか？

:::message
この記事では、TypeScript 5.1現在の状況について説明します。あなたがこの記事を読んでいる時点では状況が変わっている可能性があります。
:::

実は、TypeScriptの熟練者であっても`module`オプションを一言で説明することは難しいはずです。なぜなら、そもそもこの`module`オプションが複数の異なる意味で使われており、もはや一言で説明できるようなものではなくなってしまったからです。

この記事では、TypeScriptのメンテナーが書いた次のGitHub issueをベースに、`module`オプションを取り巻く状況を説明します。

https://github.com/microsoft/TypeScript/issues/55221

## `module`オプションの意味とは

昔は`module`オプションの意味は明確でした。昔というのは、`module`オプションの候補が`commonjs`, `amd`, `system`, `umd`, `es2015` くらいだった頃のことです。この頃は、`module`オプションは、TypeScriptが出力するモジュールの形式を指定するオプションでした。

つまり、TypeScriptのソースコードでは`import`や`export`を使ってモジュールを定義し、TypeScriptのコンパイラがそれを`module`オプションで指定した形式に変換する、という仕組みでした。`module`が`commonjs`であれば、`import`は`require`に変換され、`export`は`module.exports`に変換されます。`module`が`es2015`であれば、`import`は`import`のまま、`export`は`export`のままになります。

ちょっと雲行きが怪しくなったのは、`module: es2022`が追加されたときです。ES2022の新機能の一つに、top-level awaitがあります。これはモジュールのトップレベル（他の関数の中ではない部分）に`await`を書けるようにする機能です。TypeScriptコードでtop-level awaitを使うためには、`module`オプションを`es2022`に設定する必要があります（Node.jsでもtop-level awaitがサポートされているため、`node16`などに設定しても良いです）。

`module: es2015`などの設定の場合は、top-level awaitを使うとコンパイルエラーになります。なぜなら、top-level awaitをES2015やCommonJSなどのモジュールシステムに翻訳することができないからです。

ここで、`module`オプションは、ただ単に翻訳先のモジュールを指定するものではなくなりました。加えて、TypeScriptコードで何を書いていいのかを制御する役割も持つことになりました。

とはいえ、この段階ではまだ`module`オプションはシンプルに理解することができます。翻訳しろと言われてもできないものはできないのだから、その時はコンパイルエラーにするしかありません。

### `node16`系オプションの登場

状況が大きく変わったのは、`module: node16`と`module: nodenext`が追加されたときです。

Node.jsの特徴は、CommonJSとES Modulesの両方に対応していることです。`module: node16`などを指定した場合はTypeScriptもこれに準じた振る舞いをします。

- `.cts`ファイルは`.cjs`ファイルにトランスパイルされる。ES Modulesの構文で`.cts`ファイルを書くとCommonJSに変換される。
- `.mts`ファイルは`.mjs`ファイルにトランスパイルされる。ES Modulesの構文で`.mts`ファイルを書くとES Modulesのままになる。
- `.ts`ファイルは`.js`ファイルにトランスパイルされるが、どちらのモジュールシステムになるかはpackage.jsonの`type`フィールドによって決まる。

特定のモジュールシステムを指定していた従来の`module`オプションとは異なり、`node16`系オプションは「Node.jsに準拠」という一段階抽象化された意味を持っています。その実は、拡張子を見たり必要に応じてpackage.jsonを見に行ったりといった複雑な要件を含んでいます。

このような`module`オプションの新しい意味づけを、冒頭のissueでは次のように表現しています。

> A declarative description of the module system that will process your emitted code at bundle-time or runtime
>
> （拙訳）ランタイムに（またはバンドル時に）コードを処理するモジュールシステムを宣言するもの

つまり、例えば`node16`であれば、TypeScriptのコンパイラに伝えるのは「このコードはNode.jsで動かす」ということであり、TypeScriptはそれに合わせてチェックやトランスパイルをする、ということです。

一方で、この新しい説明は従来のオプション（特に`es2022`など）とはマッチしていません。従来のオプションはあくまで構文を指定するものであり、どのようなランタイムで動かすかを指定するものではないからです。そのため、`es2022`という明らかにES Modulesを指す値であっても、これは「ES Modules**だけ**をサポートするランタイムで動かす」というような意味ではありません。従来、`module: es2022`のコードはNode.js用だったりブラウザ用だったり、あるいはバンドラに食わせる用だったりしました。そのため、`module: es2022`は「どのようなシステム上でコードを動かすのか」を表現していないことになります。

ここで、`module`オプションが似て非なる2つの意味で使われることになりました。

## `module: node16`であらわになったCJSとESMの問題

`module: node16`は、CJSとESMの**両方**を同時にサポートするシステムです。実は、従来は両者の違いをそこまで真剣に取り扱う必要がありませんでした。いつぞやにTypeScriptに`esModuleInterop`が実装されて以降は、CommonJSとES Modulesの違いは大体うまく吸収されるため、細かいことを考えなくてもおおよそ何とかなったのです。ランタイムの側も、webpackをはじめとするバンドラがうまくやってくれていたため、CommonJSとES Modulesの違いを意識する必要はあまりありませんでした。しかし、Node.jsのモジュールシステムに正確に対応するためには、そのような雑な対応がまかり通らなくなってきました。

例えば、Node.jsではCommonJSモジュールからES Modulesを`require`することができません。TypeScriptもこの判定をサポートしています。`.cts`ファイルから`.mts`ファイルを`import`しようとすると次のようなコンパイルエラーになります。

```
The current file is a CommonJS module whose imports will produce 'require' calls; however, the referenced file is an ECMAScript module and cannot be imported with 'require'. Consider writing a dynamic 'import("./mts.mjs")' call instead.
```

逆に、ES ModulesからCommonJSモジュールを`import`することはできます。この場合、CommonJS側の`module.exports`がdefault exportと見なされます[^note_commonjs_analysis]。

[^note_commonjs_analysis]: 加えて、静的解析がうまくいけば`exports`のプロパティがnamed exportとして扱われるかもしれないとされています。

### `.mts`から`.cts`を読み込むと？

`.cts`は、TypeScriptのコードとしてはES Modulesで書けるがトランスパイル後はCommonJSになるという挙動を持ち、ファイル単体では`module: commonjs`のような動きとなります。トランスパイル例を見てみましょう。

```ts
// a.cts
export const foo = 3;
export default 123;
// ↓↓↓ トランスパイル ↓↓↓
// a.cjs
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
exports.foo = void 0;
exports.foo = 3;
exports.default = 123;
```

defaultエクスポートに着目すると、`exports.default`としてエクスポートされていることが分かります。これは、ES Modulesのdefaultエクスポートがもともと「`default`という名前でnamed exportする」のと等価な機能であることを鑑みれば、妥当な挙動です。

では、これを`.mts`ファイルから`import`してみましょう。

```ts
// b.mts
import a from "./a.cts";

console.log(a);
```

これを実行すると何が表示されるでしょうか。実は、123ではありません。次の結果が表示されます。

```
{ foo: 3, default: 123 }
```

これは前述のNode.jsの挙動に準拠しています。つまり、`.cjs`の`exports`オブジェクトがCommonJSモジュールのdefault exportとして扱われます。`.cts`で`export default`したものが`.mts`のdefault importにちょうど対応していないというのは奇妙ではありますが、Node.jsの仕様に準拠することを前提にすると、この挙動にするしかありません。

:::details .ctsから.ctsを読み込んだ場合は？

ちなみに、`module: node16`環境において`.cts`から`.cts`を読み込んだ場合の挙動は`module: commonjs`の場合と同じです。

```ts
// a.cts
export const foo = 3;
export default 123;
// b.cts
import a from "./a.cts";
console.log(a);
```

この場合、直感通り、`a`は`123`となります。`b.cts`のトランスパイル結果は次のようになっており、`esModuleInterOp`由来のコードが見えます。

```ts
// b.cjs
"use strict";
var __importDefault = (this && this.__importDefault) || function (mod) {
    return (mod && mod.__esModule) ? mod : { "default": mod };
};
Object.defineProperty(exports, "__esModule", { value: true });
const a_cjs_1 = __importDefault(require("./a.cjs"));
console.log(a_cjs_1.default);
```

:::

### CJSとESMと型定義ファイル

TypeScriptの機能には、**型定義ファイル**があります。特にライブラリをnpmなどで配布する場合。トランスパイルした`.js`ファイルと、それに対応した型定義（`.d.ts`など）を配布することが多いでしょう。

まず、`module: node16`において、`.cts`からどのような型定義ファイルが生成されるかを見てみましょう。

```ts
// a.cts
export const foo = 3;
export default 123;
// ↓↓↓ 型定義生成 ↓↓↓
// a.d.cts
export declare const foo = 3;
declare const _default: 123;
export default _default;
```

これを別パッケージの型定義として読み込むことを考えます。試しに、こんな感じでambient moduleとして宣言してみます。

```ts
declare module "cjs-module" {
  export const foo = 3;
  const _default: 123;
  export default _default;
}
```

これを`.mts`から`import`してみます。

```ts
// user.mts
import a from "cjs-module";
console.log(a);
//          ^ a: 123
```

試してみると分かりますが、`a`の型は`123`です。つまり、プロジェクト内の`.cts`をimportした場合と、ambient moduleとして定義されたモジュールをimportした場合とで、まったく同じ型定義を有するにもかかわらず、解釈が異なるということです。

ここでの根本的な問題は、CommonJSにトランスパイルされるTypeScriptプログラムであっても、それに対応する型定義はES Modulesの構文で書かれたままであり、ランタイムのモジュールシステムがどちらなのか型定義だけ見ても判別できないということです。前述のissueには次のような嘆きが書かれています。

> This is perhaps, historically, because we had no idea what the actual module format of the JS file described by the declaration file is. (It would have been really nice for declaration emit to have always encoded the output module format, but here we are.)
>
> （拙訳）これはおそらく、歴史的には、型定義ファイルが表すJSファイルの実際のモジュールフォーマットが分からないからです。（型定義を出力する際に最初からモジュールフォーマットの情報も含めておけばよかった。でも、そうはならなかった。ならなかったんだよ、ロック）

Node.jsのESMからCommonJSを読み込んだときの挙動をサポートするためには「CommonJSを読み込んだか」という判定が必要であり、その情報は現状では型定義本体に含まれていないのです。

### ちゃんとpackage.jsonがある場合

では、別のライブラリの型定義を読み込む場合は問題が起きないのでしょうか。実は、ちゃんとpackage.jsonを用意すると大丈夫です。

```json
// node_modules/cjs-module/package.json
{
  "name": "cjs-module",
  "types": "./index.d.cts"
}
```

```ts
// node_modules/cjs-module/index.d.cts
export declare const foo = 3;
declare const _default: 123;
export default _default;
```

このようにnode_modulesの中にインストールされた`cjs-module`を用意して、`.cts`ファイルを型定義ファイルとして指定します。これを`.mts`から`import`してみます（node_modules内からモジュールを読み込むために、 `moduleResolution: node16`が必要です）。

```ts
// user.mts
import a from "cjs-module";
console.log(a);
```

こうすると、`a`の型は`123`ではなく`{ foo: 3; default: 123; }`となります。つまり、`.mts`から`.cts`を読み込んだ場合と同じ挙動になります。

要するに、`module: node16`が指定してあれば、たとえnode_modules内のモジュールであっても、TypeScriptは型定義ファイルが`.cts`か`.mts`か（あるいはpackage.jsonの`type`フィールド）によってランタイムがCommonJSかES Modulesなのかを判断してくれるということです。

:::message
実際にはこの挙動を制御するのは`module`ではなく`moduleResolution`だったりするのですが、このあたりの正確な事情は近いうちに変わりそうです。そのため、ここでは踏み込んだ説明は避けておきます。

https://github.com/microsoft/TypeScript/pull/54788
:::

### バンドラとの関係は？

さらに頭が痛い問題は、少し前に追加された`moduleResolution: bundler`です。これはバンドラが行うモジュール解決を再現したオプションです。大雑把な特徴としては、Node.jsと同様にpackage.jsonの機能（`exports`など）をサポートする一方、Node.jsとは異なり拡張子の省略が許されます。

一口にバンドラと言ってもさまざまなものが存在します。そして、やはり異なるバンドラは異なる挙動をするものです。特に問題となるのはこの記事ですでに説明した「ESMからCommonJSを読み込んだときの挙動」であり、Node.jsの挙動を再現しているのか、していないのかで派閥が分かれています。

現状の`moduleResolution: bundler`はNode.jsの挙動を再現しないほうのバンドラに合わせた挙動になっているため、再現するほうのバンドラ（具体的にはwebpackとesbuild）に対応できていません。この問題を扱っているのが次のissueです。

https://github.com/microsoft/TypeScript/issues/54102

記事執筆時点でのマイルストーンはTS 5.3となっていますが、難航している印象です。

## 型定義とエコシステムの問題

これまでに説明した通り、`module: node16`ではESM（`.mts`）からCJS（`.cts`）を読み込んだときの挙動をNode.jsに合わせるという挙動が実装されました。そうなると、`module: node16`より前に作られて公開されたたくさんのパッケージの型定義は大丈夫なのかという心配が生まれます。

実は、型定義の生成をちゃんとTypeScriptでやっていたのであれば、意外と大丈夫です。TypeScriptは互換性の維持を頑張っているので、Node.jsがESM対応する前に作られたTypeScript製のパッケージなどは大体正しく認識できます。

どちらかというと問題なのは、型定義を手で書いていたり、package.jsonの書き方を間違えたりした場合です。後者はいわゆるデュアルパッケージをやろうとして間違えると起こりがちですね。そこで、このような問題に対応するために生まれたのが、arethetypeswrongです。

https://github.com/arethetypeswrong/arethetypeswrong.github.io

このツールではnpmで公開されている型定義を検査し、問題がないか調べることができます。例えばこのツールは[Masquerading as CJS](https://github.com/arethetypeswrong/arethetypeswrong.github.io/blob/4786ef2571de5772cece4c671847456b9746437e/docs/problems/FalseCJS.md)という問題を検出できます。これは、ランタイムにimportで読み込まれるのはESMなのに、型定義はTypeScriptからCommonJSとして認識されるという問題です。主に、package.jsonの書き方が良くないと発生します。

他にも、この記事の話題ととくに関連しているのが[Incorrect default export](https://github.com/arethetypeswrong/arethetypeswrong.github.io/blob/4786ef2571de5772cece4c671847456b9746437e/docs/problems/FalseExportDefault.md)という問題です。

純CommonJSのモジュールにおいて、`require()`の結果として得られるのは、読み込まれたモジュールの`module.exports`です。そのため、例えばrequireの結果が関数であって欲しければ、`module.exports = function() { ... }`のようなコードを書くことになります。

ライブラリがもともとJavaScriptで書かれていて手書きの型定義を付け足したい場合はどうするでしょうか。まず思いつくのは次のように関数を`export default`するような型定義ではではないでしょうか。

```ts
export default function(): void;
```

このようにするのは実は間違いであり、これがIncorrect default exportという問題です。というのも、CommonJSファイルに対して書かれた型定義における`export default`というのは常に`exports.default`の型を定義しているのであって、`exports`自体の型を定義しているわけではないからです。

このミスは従来のTypeScriptでは（特に`esModuleInterOp`が実装されてからは）顕在化しませんでしたが、CommonJSとESMの混用を排した`module: node16`では問題になります。古いパッケージが`module: node16`で動かなくなるとしたらこのパターンが多いでしょう。

ちなみに、この場合の正しい型定義は、`export =`構文を用いて`exports`自体の型を定義するものです。

```ts
const _default: () => void;
export = _default;
```

## `.cts`とか`.mts`の`module: node16`以外での扱い

TypeScriptは、Node.jsの`.cjs`と`.mjs`に対応するものとして`.cts`と`.mts`を導入しました。そうなると、`module: node16`以外のときに`.cts`や`.mts`をどう扱うかという問題が生まれます。

実は、今のところあまりうまい取り扱いにはなっていません。というのも、CJSとESMを区別する取り扱いは`module: node16`特有のものであるため、それ以外の設定ではこれらは拡張子が違うだけでただのTSファイルです。

そのため、例えば`module: esnext`下で`.cts`ファイルを使うと、[トランスパイル結果が`.cjs`なのに`export`構文が混ざっていたり](https://github.com/microsoft/TypeScript/issues/50647)、逆に`module: commonjs`下では[CommonJS構文で書かれた`.mjs`ファイルが出力されたり](https://github.com/microsoft/TypeScript/issues/54573)など望ましくない結果が現れてしまいます。

## まとめ: `module`オプションの今後

ここまでの話をまとめると、`module: node16`および`module: nodenext`の導入により、`module`オプションの意味が曖昧になってしまいました。`module: node16`はTypeScriptコード（をトランスパイルしたJS）がNode.js上で動くようにチェックするという意味で、従来の`module: commonjs`や`module: es2022`などとは意味合いが異なります。

Node.js対応に伴ってCommonJSとESMの共存という概念が生まれ、それによって発生した問題もありました。具体的な問題としては、バンドラによってNode.jsの模倣度合いが違うという問題があります。このようなバンドラに対応するためには、既存の`module: node16`と`moduleResolution: bundler`のどちらもうまくはまらないという問題もあります。

つまり、既存のオプション体系では十分な問題解決が難しくなってきており、見直しが必要です。

冒頭で紹介したissueでは、理想的な修正としては次の内容が挙げられています。

- `module`オプションにひとつの一貫した意味を与えるべきである。
- `module: node16`と`module: nodenext`以外の`module`は`.cts`と`.mts`の扱いが良くないので、全部非推奨にする。
- Node.jsの挙動を模倣するバンドラと模倣しないバンドラの両方に対応できるようにする。
- （将来的には、WebブラウザなどCommonJSを全くサポートしない環境を表す`module`オプションを作る）

つまり、`node16`のように`module`はランタイムの特性を表すという方向性にシフトしつつ、TypeScript本体がCJSとESMの区別を認識することを前提に`module`オプションを作り直すということになります。`module`が持つ従来の選択肢は、CJSとESMの区別がない（≒ランタイムの特性を考慮に入れていない）ため非推奨になります。

issueでは、ここに書いた以外にもさまざまな可能性が挙げられており、今後議論されることになるでしょう。TypeScriptのアップデートがあった際は、この記事で得た知識を振り返ってみるとより理解が深まるかもしれません。
