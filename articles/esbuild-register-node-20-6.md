---
title: "機能が無いことに依存していた例　esbuild-register編"
emoji: "🏢"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nodejs"]
published: true
---

皆さんこんにちは。ライブラリ等においては、機能追加は破壊的変更としては扱わないのが普通です。通常、新しい機能がライブラリに追加されても、その機能を使っていない既存コードは影響を受けません。

しかしながら、「機能が無いこと」に依存しているユーザーにとっては単なる機能追加も破壊的変更である、と（もちろん冗談で）言われることがあります。今回は、その実例を紹介します。

## Node.js 20.6.0で動かなくなったesbuild-register

[esbuild-register](https://github.com/egoist/esbuild-register)は、Node.jsでTypeScriptファイルを実行できるようにするためのパッケージです。正確には、モジュールの読み込みにフックし、[esbuild](https://esbuild.github.io/)を用いてJavaScriptにトランスパイルするという仕組みになっています。

見出しの通りesbuild-registerはNode.js 20.6.0で動かなくなってしまったのですが、実はあらゆる場面で動かなくなったわけではありません。Node.jsのモジュール読み込みはCJSとESMの2系統があり、ESMの側のフックが動かなくなってしまいました。これは、esbuild-registerのREADMEで次のように説明されているものです。

> ## Experimental loader support
> When using in a project with `type: "module"` in `package.json`, you need the `--loader` flag to load TypeScript files:
>
> ```sh 
> node --loader esbuild-register/loader -r esbuild-register ./file.ts
> ```

このように、`esbuild-register/loader`として提供されていた部分が動かなくなってしまいました。

:::details Experimentalって書いてあるけど？

上の引用部分をよく読むと、Experimentalと書いてあるのが目につきます。通常、実験的な機能は安定しているとはみなされず、メジャーアップデートではなくても壊れうるものと考えられます。

実際、ここで使われているNode.jsの`--loader`オプションも実験的機能です。そのため、マイナーアップデートで壊れたとしても取り立てて騒ぐことではないようにも思えます。

しかし、今回の壊れ方の中身まで見ると、実験的機能だから壊れたというより、機能追加によって壊れたと言っても過言ではないように思えます。そこで、この記事では「機能が無いこと」に依存していた例として紹介します。

ちなみに、このESM向けフックの機能は、Node.js 20.6.0で「Release Candidate」という安定一歩手前の状態に昇格しました。

:::

動かなくなったことは[こちらのissue](https://github.com/egoist/esbuild-register/issues/96)で報告されています。このissueでは、esbuild-registerを通しているのに「SyntaxError: Cannot use import statement outside a module」というエラーが出てしまうということが報告されています。このエラーメッセージは、`import`構文がモジュールの外（＝CommonJSモジュール）で使われているときに出るものです。うまく`import`構文がトランスパイルできていないのでしょうか。

## 動かなくなったコード

[今回問題が発生したesbuild-registerの実装](https://github.com/egoist/esbuild-register/blob/311a1ef067f0078faa870dcb6db6f29fa4ea61d1/src/loader.ts)は、とてもシンプルです。以下に全部引用します。

```ts
// https://github.com/egoist/esbuild-register/issues/26#issuecomment-1173015785

const extensionsRegex = /\.(ts|tsx|mts|cts)$/

export async function load(url: any, context: any, defaultLoad: any) {
  if (extensionsRegex.test(url)) {
    const { source } = await defaultLoad(url, { format: 'module' })
    return {
      format: 'commonjs',
      source: source,
    }
  }
  // let Node.js handle all other URLs
  return defaultLoad(url, context, defaultLoad)
}
```

一見すると何をやっているのかよく分かりませんね。（Node.js 20.6より前の）このコードの仕組みを解説します。

この`load`フックは、モジュールから他のファイルが`import`によって読み込まれる場合に呼び出されます。Node.jsでは、ESMからESMを読み込むことも、CJSを読み込むこともできます。`load`フックの返り値の`format`によって、読み込まれたモジュールがESMなのかCJSなのかを表すことができます。

このコードでやっていることは、`url`がTypeScriptファイルだった場合は問答無用で`format`を`'commonjs'`にしてからNode.jsに渡すというものです。この場合、Node.jsはCommonJSのモジュールローダーを使ってモジュールをロードします。`esbuild-register`はCommonJS向けのモジュールローダーにもフックしているので、読み込まれたファイルは無事にトランスパイルされてNode.jsに読み込まれます。

この時、`format: 'commonjs'`と一緒に返された`source`は無視されます（CommonJSのモジュールローダーが読み込み直します）。

おおよそ次のようなイメージです。

- モジュール「`foo.ts`を`import`したいです」
- Node.js「`foo.ts`ですか。`import`だから、ESMローダーで読み込みますね」
- ESMローダー「どれどれ。あ、このファイルはCommonJSですね」
- Node.js「CommonJSですか。じゃあCommonJSローダーで読み込みますね」
- CommonJSローダー「`foo.ts`を読み込みま」
- `esbuild-register`「トランスパイルするぞ」
- CommonJSローダー「した（トランスパイル済）」

## なぜNode.js 20.6.0で動かなくなったのか

以上の実装がNode.js 20.6.0で動かなくなった理由は、**`load`フックから`format: 'commonjs'`が返されたときに、`source`を使う機能が追加された**からです。この機能追加により、CommonJSの読み込みに対しても、`load`フックのようなAPIで処理を完結させられます。従来のCommonJS向けのフックの実装は、Node.jsの内部実装を直接モンキーパッチするような行儀の悪いものでした。Node.js 20.6.0の新機能によりモンキーパッチをする必要はなくなり、CommonJSに対してもより安定した、公式のAPIを使えるようになったということです。

言い換えると、従来は`load`フックは「このファイルはCommonJSだよ」と教える役割しか無く、実際の読み込みは従来のCommonJSローダーがやるしかありませんでした。Node.js 20.6.0では「このファイルはCommonJSで、中身はこうだよ」と教える役割になり、従来のCommonJSローダーを呼び出す必要が無くなったということです。

そして、`esbuild-register`の実装においてはこれまで無意味に`source`を返していました。これは`defaultLoad`の結果なので、トランスパイルなどの処理は通っていません。そして、Node.js 20.6.0からはこの返された`source`がロード結果として扱われるようになったため、トランスパイルされていないものがNode.jsに渡されてしまったことになります。

- モジュール「`foo.ts`を`import`したいです」
- Node.js「`foo.ts`ですか。`import`だから、ESMローダーで読み込みますね」
- ESMローダー「どれどれ。あ、このファイルはCommonJSですね。読み込んでおきましたよ（未トランスパイル）」
- Node.js「どうも。じゃあ実行しま（SyntaxError）」

[実際のNode.jsの実装](https://github.com/nodejs/node/pull/47999)としては、ESMローダーの中に新たにCommonJSとの互換層みたいなものが作られたイメージです。この新たな互換層は、`format: 'commonjs'`とともに`source`を返すことでオプトインできます。

そして、この互換層は従来のCommonJSローダー（リンク先の言葉を借りれば“monkey-patchable”なローダー）とは無関係なので、トランスパイルが挟まる機会がありません。

今回、`esbuild-register`は、意図せずNode.jsの新機能にオプトインしてしまったことで、動かなくなってしまったのです。

## 修正方法

考えられる修正方法は簡単です。`source`を返すのをやめれば従来の挙動に戻り、動くようになるでしょう。

ただ、実はこれは長期的に望ましい解決策ではありません。なぜなら、Node.jsのドキュメントで次のように言及されているからです。

> If `source` is undefined or `null`, it will be handled by the CommonJS module loader and `require`/`require.resolve` calls will not go through the registered hooks. This behavior for nullish `source` is temporary — in the future, nullish `source` will not be supported.

つまり、`source`を返さない場合のサポートは後方互換性のための一時的な対応であり、将来的には`load`フックが`source`を返すことが必須になるということです。そのため、`esbuild-register`は将来的には`source`を返す方法で修正する必要があります。

## 余談

筆者は[nitrogql](https://github.com/uhyo/nitrogql)の開発中にこの問題に直面しました。設定ファイルなどがTypeScriptで書かれていても読み込めるようにするために、`esbuild-register`を使っていたのですが、Node.js 20.6.0で動かなくなってしまったのです。

筆者がとった解決策としては、[esbuild-register相当のものを再実装しました](https://github.com/uhyo/nitrogql/tree/master/packages/esbuild-register)。

`esbuild-register`にコントリビュートすることももちろん考えるべきですが、スピード感の問題や、起こりうる互換性の問題全てに責任を持てないためPRを出すのは見送りました（代わりに、この記事に書いたような分析を簡単にまとめてissueにコメントしています）。

他に、`esbuild-register`は、`"type": "module"`の環境など、ESMとして書かれたTypeScriptモジュールでも強制的にCommonJSとして取り扱って読み込むという仕様になっていました。これは長期的に見て良くないのではないかと考えたため、筆者による実装では、ESMとして書かれたモジュールはトランスパイル後もESMのままで、Node.jsのESM実装によって実行される仕様としました。

ESMはESMのままにしておいた方がフックの役割が薄くなるので、望ましいと考えています。例えば、`esbuild-register`はESMがCJSに変換されるケースがあるため、`import.meta.url`をエミュレーションする機能が入っています。筆者の実装ではESMはESMになるので、`import.meta.url`は変換する必要がありません。

本家の`esbuild-register`もいずれはそのようになることを期待しつつ、それをやるのは自分には荷が重すぎるので今回は独自実装にしました。Node.js 18系のサポートが終了すれば、新しいモジュール読み込みフックに移行することで従来のモンキーパッチを利用した実装を辞めることができるので、そのタイミングで実装を大きく変えるのがよいでしょう。

## まとめ

今回は、「機能が動かないことに依存していた人にとっては機能追加は破壊的変更」という故事（？）の実例を紹介しました。