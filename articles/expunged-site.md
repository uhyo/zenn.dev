---
title: "Cloudflare PagesとCloudflare Workersで██された██を作る"
emoji: "🕶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "javascript"]
published: true
---

みなさん█████。みなさんは、許可した人にだけ███████けどそれ以外の人にも██████を██して██████たいと思ったことがあるのではないでしょうか。

この記事では、そのような機構をCloudflareで実現してみたのでご紹介します。█████以外は全部コードが公開されており、皆さんも簡単に自分でこのような██を作ってCloudflareにデプロイすることができます。

この記事の末尾には、█████可能なトークンを用意してあります。[デモサイト](https://expunged-site.uhyo.workers.dev/) にアクセスしてトークンを入力してみてください。

# 仕組み

まず、サイト自体は静的サイトジェネレーターである████████を用いて作成します。このとき、██がされたバージョンとされていないバージョンの2種類のアウトプットを用意し、後者は`/__secret__`以下に██します。

このサイトは普通にCloudflare Pagesにデプロイしますが、その前段にCloudflare Workerを配置し、そちらのURLを██します。これにより、このサイトへの████は全てワーカーを経由するようになります。

お察しの通り、このワーカーが██されたサイトと██されていないサイトの████を行います。ワーカーは████████として渡されたトークンを見て████の████があるかどうか判断します。トークンとしては、今回は簡単に扱える███を採用しました。簡単な███である割に、████などの仕組みが組み込まれており便利です。

ワーカーは基本的には████████（Cloudflare Pages）への████として振る舞いますが、有効なトークンが与えられたときは███を修正し`/__secret__`を付加します。こうすることで、有効なトークンが与えられた場合は█████されたページを返すことができます。直接閲覧されないように、`/__secret__`で始まるURLにアクセスした人は██████します。

以下では、各部分について解説します。

# 静的サイトのビルドとデプロイ

上述のように、静的サイトジェネレータとしては████████を使用し、それでビルドされたサイトをCloudflare Pagesにデプロイします。このとき、いくつかの████があります。1つは█████の維持、もう1つは文章の████の管理です。

## ████████での███████読み込み

まず1つ目については、サイトの趣旨上、███████はリポジトリに含めたくありません。しかし、████版サイトのビルドもCloudflareのビルドノード上で行われるため、███████は██████としてCloudflare Pagesに与えられる必要があります。

その具体的な方法は████です。そのため、██の中身も████として与えやすい形で管理する必要があります。実際には、このシステムでは`.env`ファイルに次のような形で、1つの変数に複数の███████が1行1個の形で管理されています。

```
TEXT1="
こんにちは
情報を見せたい
断片的な文書
公開
"
```

これを読み込む側は次のようにします。環境変数も████████では████████として扱うのがよいでしょう。上のような█████の文字列はこのファイルで配列に変換されます。後述しますが、Cloudflareのダッシュボード上では残念ながら█████████ため、SEPARATORを別の文字に変えられる仕組みになっています。

```js
// src/_data_secretText.js
const dotenv = require("dotenv");

module.exports = () => {
  dotenv.config({ override: true });
  return loadSecretData();
};

function loadSecretData() {
  const { SEPARATOR = "\n" } = process.env;
  const chunks = [];
  for (let i = 1; ; i++) {
    const chunk = process.env[`TEXT${i}`];
    if (chunk !== undefined) {
      chunks.push(chunk.trim());
    } else {
      break;
    }
  }
  return chunks.join(SEPARATOR).split(SEPARATOR);
}
```

コードをよく読むと分かるように、████は`TEXT1`, `TEXT2`, ……のように█████することができます。これは一つの████の長さには制限があるため、それを克服するためです。

## ████████での利用

では、このデータを使う側はどのようにするのでしょうか。

今回のサンプルではテンプレートエンジンとして████████を使っていますが、文書の██████は次のような具合です

```html
<p>みなさん{% expunged %}。みなさんは、許可した人にだけ{% expunged %}けどそれ以外の人にも{% expunged %}を{% expunged %}して{% expunged %}たいと思ったことがあるのではないでしょうか。</p>
```

このように、████████は{% expunged %}として表現します。一般の██████などではそれぞれの箇所に███が必要になるところですが、今回は極力████にするために███は廃しました。お察しの通り、前述のデータが████に███████形になります。

{% expunged %}の実装は████████████に書くことができます。

```js
  eleventyConfig.addShortcode("expunged", function () {
    const cnt = counter.increment(this.page.date);
    const text = this.ctx.secretText[cnt] || "";
    if (SECRET) {
      return `<span class="revealed">${text}</span>`;
    } else {
      return `<s class="expunged">${"█".repeat([...text].length)}</s>`;
    }
  });
```

ただしcounterというのは次のKeyedCounterクラスのインスタンスです。

```js
class KeyedCounter {
  currentKey;
  count = 0;
  initialValue;
  constructor(initialValue) {
    this.initialValue = initialValue;
    this.count = initialValue;
  }
  increment(key) {
    if (this.currentKey !== key) {
      this.count = this.initialValue;
      this.currentKey = key;
    }
    return this.count++;
  }
}
```

このように、実装は現在███████かどうかを判断し、それに応じて生の████を描画するか███████を描画するかを決めます。後者の場合も文字数だけは開示するようにしています。

前述のように、█████は████で表現しています。この場合、██████████████████という問題がありますが、それに対処できるように仕組みを修正するのは████████ので、███████とします。

また、コードでは`this.page.date`が参照されており、これが変わると████が████されます。これは、████████を████████で使用する際もこの仕組みが正しく動作するために必要です。`this.page.date`は████████████であり、これが変わったら█████████することで、再ビルド時も結果が正しくなります。

# Cloudflare Workersの実装

こちらは███████████。概要は前述の通りで、████らしいものは██████くらいです。███████████は██していますので、詳細は███████████。

ワーカーは█████である必要があるようですが、公式ではそのために███████を用いて████することが推奨されています。Cloudflare Workersのエミュレータであるminiflareでも、watchモードで██████████時に████できるようになっており、███████と相性のいいワークフローになっています。

# ███
この記事では、Cloudflareのスタックを用いて███████████を作成する方法をご紹介しました。これまでのネタとは異なり、今回は████により█████できる造りになっています。█████以外は下記の███████████で全て見ることができますので、みなさんも自分の█████を作ってみましょう。今回はデモサイトのみを用意しましたが、筆者は将来的には自身のウェブサイト uhy.ooo にこれを導入しようかなと思っています。

現状では██████と████を███████████のが難点で、█████となっています。

https://github.com/uhyo/expunged-site

---

ひみつのアクセスコードを[デモサイト](https://expunged-site.uhyo.workers.dev/)で入力してみよう！

```
eyJhbGciOiJIUzI1NiJ9.eyJjbGVhcmFuY2UiOjEsImV4cCI6MTY4MDI3NDc5NSwiaWF0IjoxNjQ4NzM4Nzk1fQ.BtA02bCxVdVPgWgDyVIsCHLzKItiTjqc47WtHatCnW8
```