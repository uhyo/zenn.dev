---
title: "TS 5.7の --rewriteRelativeImportExtensions オプションを使う前に読む記事"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

TypeScript 5.7で追加される `--rewriteRelativeImportExtensions` オプションは、その使用にあたって注意が必要なオプションです。

:::message
この記事の公開時点ではこのオプションはまだ正式リリースされていませんが、すでに実装がマージされているため次のリリース（TS 5.7）で利用可能になると思われます。
:::

背景としては、このオプションに関して最近英語圏のTSエヴァンジェリストのような人が積極的な活用を推奨する[投稿](https://x.com/mattpocockuk/status/1840747531740844509)をしました。一方で、TypeScriptチームはこのオプションを使うのは限定的な場合に限るべきとしています。

この記事ではTypeScriptチームの側に寄り添い、`--rewriteRelativeImportExtensions` オプションをむやみに使うべきではない理由について解説します。

以下に引用するのはTypeScriptチームのRyan氏の投稿のひとつです。

https://x.com/SeaRyanC/status/1840815156894617652?conversation=none

> If you can't coherently explain why this flag wasn't present for the previous 30 iterations of TypeScript, and what changed, don't turn it on

**意訳**: このフラグがなぜこれまでの30回のTypeScriptのリリースで追加されなかったのか、それが今になって追加された理由が何なのか、それを理路整然と説明できないのであれば、このフラグを有効にしないでください。

ということで、どうしても`--rewriteRelativeImportExtensions` オプションを使いたい人は、この記事を読んで理路整然と説明できるようになりましょう。

## 結論

Node.jsの`--experimental-strip-types`オプションを使うならこのオプションを有効にしていいです。

## 背景

TypeScriptのNode.js (ESM) 向けモジュール解決では、import宣言で相対パスを使う場合には、拡張子として`.js`を使う必要があります。実際のTypeScriptプロジェクトでは`.ts`といった拡張子のファイルが使われますが、それでもimport宣言には`.js`を使う必要があります。

```ts
// src/foo.ts
export const foo = 'foo';

// src/bar.ts
import { foo } from './foo.js'; // ← 拡張子が.jsになっている
console.log(foo);
```

これは人によっては不自然に感じるもので、`.ts` を使ってimportさせてくれという要望はたびたび出ていました。

しかし、TypeScriptでは意図的にこの仕様になっていました。その理由は、**`.ts`ファイルはトランスパイルしたら`.js`になるから**です。

```
src
├── foo.ts
└── bar.ts

↓↓↓↓↓ tscでビルドすると ↓↓↓↓↓

dist
├── foo.js
└── bar.js
```

つまり、結局このimportが実行・解決されるのはランタイム時なのだから、その時にそのまま動くように`.js`を使うべきだということです。そのため、`.ts`でインポートできるという提案はこれまで受け入れられていませんでした。

TypeScriptは型定義の解決のために独自にimportの解決などを行いますが、それはあくまで型検査のためであり、ランタイムの挙動にTypeScript側が合わせに行っています。逆にTypeScriptの側からランタイムの領域（import specifierに何を書くのか）に干渉するのはTypeScriptの範疇を超えているということでしょう。

### `--allowImportingTsExtensions` オプション

とはいえ、実はTS 5.0では `--allowImportingTsExtensions` というオプションが追加されています。このオプションを有効にした場合は`./foo.ts`のようなインポートが可能になります。

ただし、このオプションの追加自体は、上記の方針が変わったことを意味しているわけではありません。

というのも、このオプションを使うには、同時に`noEmit`オプションも有効化する必要があります。

つまり、**トランスパイルして`.js`を出力しないという誓約**をしないとこのオプションは使えないということです。これであれば、上記の方針とは矛盾しません。

トランスパイルしない環境というのは、具体的にはバンドラを使う場合です。この場合、`.ts`→`.js`という変換はファイル単位では発生せず、バンドラがモジュール解決を含めて内部的に処理してしまいます。そのため、`.ts`でインポートしても問題ないのです。

言い方を変えれば、バンドラというのは`.ts`のインポートをランタイムに理解できる実行環境だと言えます[^bundler_host]。

[^bundler_host]: もちろんバンドラはJavScriptランタイムではありませんが、importの解決を事前に行うという意味では、モジュール解決において部分的にJavaScriptのランタイムの役割を担っていると言えます。

## Node.jsの`--experimental-strip-types`オプションの登場

上記の結論ですでに名前が出ていましたが、この状況を変えたのがNode.jsに実装された`--experimental-strip-types`オプションです。このオプションの詳細は別の記事に譲るとして、ここでは要点だけ説明します。

このオプションは、`node index.ts` のようにTypeScriptファイルを直接Node.jsで実行できるようになる機能です。Denoなどと同様に、与えられた`.ts`ファイルをその場でSWCでトランスパイルすることで実現されています。

ポイントは、このオプションを使う場合、**`.ts`でimportしないといけない**ということです。

```ts
// src/bar.ts
import { foo } from './foo.ts'; // ← 拡張子を.tsにする必要がある
console.log(foo);
```

これは、Node.jsにおけるモジュール解決を簡単にしたいという理由があります。つまり、従来のTSの慣習では「ファイルシステム上の`foo.ts`を指すために`./foo.js`と書く」のようなことをしているため、Node.js側で`.js`を`.ts`に読み替えてファイルを探すといった挙動をしたくなかったのでしょう。

TypeScript側から見ると、これは方針に十分な影響を与える変化です。importを`.js`で書く理由が「ランタイムでは`.js`でモジュール解決するから」だったのに、Node.jsというランタイムが`.ts`でモジュール解決できるようになったのであれば、前提が崩れることになります。

しかも厄介なのは、では「Node.js向けプロジェクトでは常に`.ts`でモジュールを解決する」と決められるのかといえば、そうではないということです。TSからJSへのトランスパイルにはオーバーヘッドがかかりますから、`--experimental-strip-types`のユースケースとして「開発時は`--experimental-strip-types`を使い、プロダクションビルドではトランスパイル済みのJSをデプロイする」という使い方が考えられます。

こうなると、「`.js`でインポートするランタイム」と「`.ts`でインポートするランタイム」の両方に**1つのコードベースで対応しなければならない**というとても厄介な状況になります。

## `--rewriteRelativeImportExtensions` オプションの追加とその困難

こうした状況を受けて、TypeScriptに`--rewriteRelativeImportExtensions`オプションが追加される運びとなりました。2種類の異なるモジュール解決をするランタイム（`--experimental-strip-types`ありのNode.jsと無しのNode.js）に対応するためには、どちらかを基準として、ビルド時に拡張子の書き換えをするしかありません。`--experimental-strip-types`のユースケースとしてビルドしないことが想定されるので、TypeScriptコード上は`.ts`として、`tsc`でのビルド時に`.js`に書き換えるのが自然です。

`--rewriteRelativeImportExtensions`オプションを使うと、この拡張子の書き換えをTypeScriptのトランスパイル時に行うことができます。

```ts
import { foo } from './foo.ts'; // ← .tsでimport

// ↓↓↓↓↓ tscでビルドすると ↓↓↓↓↓

import { foo } from './foo.js'; // ← .jsに書き換え
```

何だできるじゃん、何で今までやらなかったの、と思われるかもしれません。しかし、色々なエッジケースがあります。

例えばdynamic importを考えてみましょう。

```ts
const { default: foo } = await import('./foo.ts');

// ↓↓↓↓↓ tscでビルドすると ↓↓↓↓↓

const { default: foo } = await import('./foo.js');
```

いいですね。ではこれはどうでしょうか。

```ts
const { default: foo } = await import(getImportPath());

function getImportPath() {
  if (process.env.NODE_ENV === 'production') {
    return './foo.ts';
  } else {
    return './foo.dev.ts';
  }
}
```

そう、dynamic importではimport先を実行時に決めることができます。そのため、トランスパイル時に拡張子を書き換えればいいという幻想はここで儚く散ることとなります。

この場合に対するTypeScriptの答えはこうです（記事執筆時点のTS Nightlyの実行結果）。

```js
var __rewriteRelativeImportExtension = (this && this.__rewriteRelativeImportExtension) || function (path, preserveJsx) {
    if (typeof path === "string" && /^\.\.?\//.test(path)) {
        return path.replace(/\.(tsx)$|((?:\.d)?)((?:\.[^./]+?)?)\.([cm]?)ts$/i, function (m, tsx, d, ext, cm) {
            return tsx ? preserveJsx ? ".jsx" : ".js" : d && (!ext || !cm) ? m : (d + ext + "." + cm.toLowerCase() + "js");
        });
    }
    return path;
};
const { default: foo } = await import(__rewriteRelativeImportExtension(getImportPath()));
function getImportPath() {
    if (process.env.NODE_ENV === 'production') {
        return './foo.ts';
    }
    else {
        return './foo.dev.ts';
    }
}
```

😇

そうです、ランタイムに正規表現で拡張子を書き換える関数を埋め込むのです。静的に書き換えできない場合は何もしない（ランタイムにエラーになる）という選択肢もあったようですが、TypeScriptは最大限頑張ることを選んだようです。静的に書き換えできない場合でも、なんとか一貫した挙動を提供しようとしてくれます。`--rewriteRelativeImportExtensions`オプションを使うということは、これを受け入れるということを意味します。

他のエッジケースもあります。

```ts
import foo = require('./foo.ts');
console.log(foo);
```

:::details この構文は？

この`import ... = require`構文はTypeScript独自の構文で、CommonJSでのみ使用可能で`require(...)`に相当するものです。

TypeScriptはランタイムにCommonJSで解釈されるファイルをESMの`import`で書くこともできますが（ランタイムに`require`にトランスパイルされます）、ES ModulesとCommonJSのセマンティクスの違いから、どうしてもESMの`import`を使えないこともあります。その場合にこの構文が役に立ちます。

現状、この構文はNode.jsの`--experimental-strip-types`ではサポートされませんが、`--experimental-transform-types`ではサポートされているので今回例に使っています。

:::

これもある種の`import`であり、`--rewriteRelativeImportExtensions`による拡張子書き換えの対象となります。

しかしこの場合、トランスパイル時に`./foo.js`に書き換えるのが不正解となるケースがあります。

それは、CommonJSでは`require('./foo.ts')`が同じディレクトリの`foo.ts`というファイルに解決されるのではなく、`foo.ts`というディレクトリの中の`index.js`というファイルに解決されるケースがあるからです。

面倒くさいですね。CommonJSまで考慮に入れると、「末尾の`.ts`を`.js`に書き換える」という単純なルールが通用しないことが分かりました。

このエッジケースに対するTypeScriptの対応は、「**`./foo.js`という誤った書き換えを実行しつつ、コンパイルエラーを出す**」です。

ファイルシステムの内容によってTypeScriptのトランスパイル結果が変わるということは受け入れがたいので、TypeScriptは上記の正規表現で書けるような機械的なルールで置き換えを行います。しかし、ファイルシステムまで加味してそれが誤りであると検知できた場合には親切にもエラーを出してくれるのです。

これはTypeScriptの既存挙動と合致していますね。TypeScriptでは、型エラーが出るようなコード（動かないと分かっているコード）でもとりあえずトランスパイル結果を出力することはしてくれつつ、型エラーを出します。それと同じと言えます。

ちなみに、逆に「書き換えないとランタイムにエラーになるけど、ルールに当てはまらないので書き換えない」というケースもあります。次のような場合ですね。

```ts
import { foo } from '#common/foo.ts';
```

[Subpath imports](https://nodejs.org/docs/latest/api/packages.html#subpath-imports)を使った`#`始まりのパスの場合、拡張子を書き換えるのが“正しい”かどうかはpackage.jsonの内容次第です。そのため、機械的なルールで判定できません。TypeScriptは一括で「書き換えない」側に倒しています。これも同様に、package.jsonの中身を見れば書き換えが正しかったかどうか判定できるので、TypeScriptは型エラーを報告してくれます。

ここまで読んでいかがでしょうか。「何だ型エラーが出るならまあいいじゃん」と思ったかもしれません。しかし、このようなエッジケースは必ず型エラーで検知できるとも限りません。そもそも、dynamic importの場合（書き換えがランタイムで行われる）と上述のエッジケースが組み合わさった場合、もう型エラーで検知するのは無理です。ランタイムエラーを受け入れるしかありません。

`--rewriteRelativeImportExtensions`は、このようなややこしいエッジケースを受け入れながら使わなければいけません。TypeScriptチームの側としても、必要だから実装したけど、今後意味不明なエッジケースがぞくぞくバグ報告で出てくるだろうと思うと頭が痛いことでしょう。

## `--rewriteRelativeImportExtensions` オプションを使うべきか

記事の最初に述べたように、TypeScriptチームは、このオプションを使うのは限定的な場合（Node.jsの`--experimental-strip-types`を使う場合）に限るべきだとしています。

ここで、Ryan氏が投稿したミーム画像を見てみましょう。

https://x.com/SeaRyanC/status/1840922680725553237?conversation=none

:::details 画像の代替テキスト
画像は上下に分かれています。上半分にはマットの上に立つ3人の女性が描かれており、「Write the path that works at runtime」とキャプションが付けられています。

下半分には6人の人間が描かれており、鉄棒、台、火が付いた車を使った、体操のように見える曲芸的なパフォーマンスをしています。こちらには「Write the wrong path, then turn on a flag to use a regex, then learn when the regex applies, then the regex makes the path right again. Separately learn if "await import" acts like static import or readFile」というキャプションが付けられています。
:::

この画像は、`--rewriteRelativeImportExtensions`オプションを使うことで物事をむやみに複雑にしてしまうことを揶揄しています。

特に、`--experimental-strip-types`と関係ない場面でこのオプションを使う場合、やっていることは「わざわざ間違った（ランタイムに解決できない）パスを書き、それをわざわざ正規表現で元に戻す」というある種本末転倒なことです。

さらに、この機能の説明を聞いたときに「それってdynamic importの場合どうなるの？」といった疑問を持つのは、ある程度のレベルのプログラマであれば自然なことです。それを調べたり、あるいは上述のようなエッジケースを理解する必要が出てきます。

結局、「ランタイムで`.js`になるからTSコード内でも`.js`で書く」という単純なメンタルモデルに比べて`--rewriteRelativeImportExtensions`の挙動は複雑であり、必要も無いのにわざわざ複雑なメンタルモデルが要求される機能を使うべきではないというのが、TypeScriptチームの立場です。

## まとめ

筆者はこの記事で述べたようなTypeScriptチームの立場を今のところ支持しています。

そのため、Node.jsの`--experimental-strip-types`などを使うわけでもない場面で`--rewriteRelativeImportExtensions`オプションを使うことは推奨しません。

皆さんも、このオプションを使う場合は、こういった背景をよく理解するようにしましょう。

:::message
**ブコメ閉鎖RTA開催中！**

このような記事を書くと不愉快なコメントがつきやすいです。以下の2点を満たすコメント（筆者の主観により判断）が筆者の目に入った場合はブコメを閉鎖します。悪しからずご了承ください。

- この記事、筆者、あるいはこの記事で言及した各種技術に対して批判的な内容である
- 批判を支える論理的な主張が無いか、あってもこの記事と関係ない主張である
:::

