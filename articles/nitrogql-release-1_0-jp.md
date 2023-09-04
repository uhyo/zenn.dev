---
title: "【日本語訳】nitrogql 1.0リリース"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["graphql", "nitrogql"]
published: true
---

この記事は、本日公開された次の記事の日本語訳です。海外で人気の記事を翻訳してみたシリーズと見せかけて、実は自分で英語で書いた記事を自分で日本語に訳して自分が作ったツールの宣伝をする記事です。元記事はこちらです。

https://nitrogql.vercel.app/blog/release-1.0

---

本日、nitrogqlの**最初の安定版**をリリースしました！

**nitrogql**はTypeScriptプロジェクトでGraphQLを使うためのツールチェインです。

## nitrogqlとは

現在のところ、nitrogqlは主に2つの機能を持ちます。それは、**コード生成**と**静的チェック**です。

### コード生成

**コード生成**は、GraphQLコード（スキーマとオペレーションのどちらも）からTypeScriptコードを生成することを指します。これにより、TypeScriptコードからGraphQLオペレーションを型安全に使用できるようになります。また、ボイラープレートコードを書く必要もありません。

これは**スキーマファースト**アプローチとして知られています。このアプローチでは、まずGraphQLスキーマを書き、それを元にコードを書きます。これとは逆のやり方がコードファーストで、その場合はあなたが書いたコードを元にスキーマが生成されます。

例えば、次のようなGraphQLオペレーションがあるとします。

```graphql
# getPosts.graphql
query getPosts {
  posts {
    id
    title
  }
}
```

これに対して`nitrogql generate`を実行後、TypeScriptコードからこのオペレーションを次のように使うことができます。

```ts
import { useQuery } from "@apollo/client";
import getPostsQuery from "./getPosts.graphql";

export const Posts: React.FC = () => {
  const { data } = useQuery(getPostsQuery);
  return (
    <ul>
      {data.posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
};
```

注目に値するのは、nitrogqlでは`.graphql`ファイルからGraphQLオペレーションをimportできることです。[TypedDocumentNode](https://github.com/dotansimha/graphql-typed-document-node)の恩恵により、このオペレーションは取得できるデータの形を知っています。これにより、変数`data`の型も元のGraphQLオペレーションから導出されます。具体的には、`{ posts: { id: string; title: string; }[]; }`となります。

最大限の安全性を得るために、nitrogqlにはwebpack用loaderとRollup用プラグインが付属しています。これにより、上のコードのように、TypeScriptコードから直接`.graphql`ファイルをimportできます。こうすることで、nitrogqlが`.graphql`ファイルのランタイムと型の両方を管理できるようになり、両者が常に同期していることを保証できます。

### 静的チェック

コード生成によってTypeScriptレベルでの型安全性が保証されますが、まだGraphQLコードにミスがある可能性を完全に排除できたわけではありません。例えば、スキーマに存在しないフィールドをqueryで取得しようとしたり、フィールドに必須の引数を渡し忘れたりすることがあります。このような、スキーマ内のエラーや、スキーマとオペレーションの不一致によるエラーは、上記のコード生成ではカバーされません。なぜなら、TypeScript向けのコード生成の際は、GraphQLレベルではコードが正しいということを前提としているからです。

ここで役に立つのが静的チェックです。静的チェックとは、TypeScriptコードを生成する前に、GraphQLコードにエラーがないかをチェックすることです。ここでGraphQLレベルでの正しさが確保され、GraphQLコードにミスがあることによるランタイムエラーを防ぐことができます。

以下は、nitrogqlがキャッチできるGraphQLレベルのエラーの例です。

```graphql
# getPosts.graphql
query getPosts {
  posts { # 引数の渡し忘れ
    id
    tite # typo
  }
}
```

これに対して`nitrogql check`を実行すると、次のようなエラーが表示されます。

```
query getPosts {
  posts {
  ^
  Required argument 'filter' is not specified

    # forgot to pass an argument
    id
    tite # typo here
    ^
    Field 'tite' is not found on type 'Post'
  }
}
```

## なぜnitrogqlを使うのか

あなたはすでに、nitrogqlと同じような機能を持つ他のツールをご存知かもしれません。特に、[GraphQL Code Generator](https://graphql-code-generator.com/)はGraphQL向けコード生成の分野で人気のツールです。

では、なぜ我々はGraphQL Code Generatorを使わずにnitrogqlを作ったのでしょうか。その理由はいくつかあります。

- Source mapが欲しいから。
- GraphQL Code Generatorの`near-operation-file` presetが好きだったから。
- 最大限の安全性をデフォルトで提供したいから。

それぞれの理由を詳しく見ていきましょう。

### Source map

Source mapは、コード生成を使う際に非常に役立つ機能です。これにより、特定のTypeScriptコードを生成した元のGraphQLコードを見ることができます。特に、エディタのGo to Definition機能を使うときに非常に役立ちます。例えば、次のコードの`getPostsQuery`にGo to Definition機能を使ってみましょう。

```ts
import { useQuery } from "@apollo/client";
import getPostsQuery from "./getPosts.graphql";

export const Posts: React.FC = () => {
  const { data } = useQuery(getPostsQuery);
  return (
    <ul>
      {data.posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
};
```

すると、生成された`getPostsQuery.d.graphql.ts`ファイルではなく、`getPosts.graphql`ファイルに移動します。これは、nitrogqlが元のGraphQLコードを指すsource mapを生成しているためです。

GraphQL Code Generatorはsource mapをサポートしていません。実は、nitrogqlを作る前にGraphQL Code Generatorにsource mapサポートを追加しようとしましたが、GraphQL Code Generatorの現在のアーキテクチャではsource mapサポートを追加するのが非常に難しいことがわかりました。source mapサポートは、コード生成ツールのコアに組み込まれている必要があり、後付けで追加するのは本当に難しいのです。そのため、私はsource mapサポートを最初から持つ新しいツールを作ることにしたのです。

### near-operation-file preset

`near-operation-file` presetは、GraphQL Code Generatorでかつて推奨されていたものです。このpresetでは1つのGraphQLオペレーションファイルに対して1つのTypeScriptファイルが生成されます。これは、GraphQLコードとTypeScriptコードを互いに分離しておくことができる非常に便利な機能です。

しかし、現在ではこのnear-operation-file presetはまだ問題なくサポートされてはいるものの、GraphQL Code Generatorのデフォルトではなくなっています。代わりに、`client-preset`がデフォルトになりました。

私はこの新しいデフォルトがあまり好きではありません。なぜなら、GraphQLコードとTypeScriptコードを同じファイルに置く必要があるからです。特に、TypeScriptコード中に書かれたGraphQLコードが解析され、型を生成され、その生成されたコードがTypeScriptコードに影響を与えます。このような循環的な仕組みはあまり好きではありません。

nitrogqlはこれとは逆のアプローチを取っています。nitrogqlでは、GraphQLコードを`.graphql`ファイルに置くことを要求し、TypeScriptコードとは完全に分離しています。これはnear-operation-file presetが行うことと似ています。nitrogqlはこのアプローチを続けていきます。もしあなたがnear-operation-file presetが好きであれば、nitrogqlも気に入るでしょう。

### 最大限の安全性

GraphQL Code Generatorはとても柔軟性のあるツールで、多くの設定項目があります。残念なことに、その中には型安全性を下げてしまうような設定項目もあります。

例えば、カスタムスカラー（GraphQLのデフォルトには存在しない追加のスカラー）の型はデフォルトでは`any`になっています。[strictScalars](https://the-guild.dev/graphql/codegen/plugins/typescript/typescript#strictscalars)をtrueにしない限り、GraphQL Code Generatorはこのことについて警告すらしてくれません。

また、GraphQL Code Generatorを使ってresolverの型定義を生成する場合、resolverを書き忘れてしまうことのを防ぐことができず、ランタイムエラーの原因となります。このことについては[別の記事](https://zenn.dev/babel/articles/graphql-typing-for-babel)で詳しく紹介しています。

nitrogqlは、デフォルトで最大限の安全性を提供するように設計されています。例えば、カスタムスカラーの型は明示的に定義する必要があり、定義しないとコンパイルエラーになります。この挙動を無効にする方法はありません。

nitrogqlにも設定項目はいくつかありますが、それらは異なるユースケースのためのものです。型安全性を下げるような設定項目はありません。

## nitrogqlを使い始める

このサイト（訳註: nitrogqlの公式サイト）には[Getting Started](https://nitrogql.vercel.app/guides/getting-started)ページがありますが、この記事でも簡単に紹介します。

nitrogqlはCLIツールです。次のようにインストールすることができます。

```sh
npm install --save-dev @nitrogql/cli @graphql-typed-document-node/core
```

`@graphql-typed-document-node/core`は生成されたコードが依存するため必要です。

次に、プロジェクトルートに`graphql.config.yaml`を作成します。例は次のとおりです。

```yaml
schema: ./schema/*.graphql
documents: ./src/**/*.graphql
extensions:
  nitrogql:
    generate:
      schemaOutput: ./src/generated/schema.d.ts
```

この設定ファイルは、nitrogqlにGraphQLスキーマを`./schema`から、オペレーションを`./src`から探すように指示します。スキーマとオペレーションは異なる構文を持つため、混在させることはできません。

次に、`nitrogql generate`を実行して、GraphQLコードからTypeScriptコードを生成します。このコマンドは各`.graphql`ファイルの隣に`.d.graphql.ts`ファイルを生成します。これらの生成されたファイルのおかげで、TypeScriptコードから`.graphql`ファイルを型安全にimportできるようになります。

また、`nitrogql check`を実行することで、GraphQLコードにエラーがないかをチェックできます。

詳しくは、[Configuration Options](https://nitrogql.vercel.app/configuration/options)ページを参照してください。

## GraphQL Code Generatorから移行する

あなたがすでにGraphQL Code Generatorを使っている場合、nitrogqlに移行したいと思うかもしれません。互換性が完璧にあるわけではないため、移行は大変な作業です。しかし、詳細な[移行ガイド](https://nitrogql.vercel.app/guides/migrating-from-graphql-codegen)をご用意しています。

## チェックだけ使う

nitrogqlを使ってみたいけど、まだGraphQL Code Generatorから完全に移行することができない場合は、nitrogqlをlinterとしてのみ使うという選択肢もあります。CIでnitrogql checkを実行するだけで、GraphQLコードにエラーがないかをチェックできます。こうすれば、完全にnitrogqlに移行することなく、静的チェックの恩恵を受けることができます。

チェックのみを行うための設定は非常にシンプルです。設定ファイルに`schema`と`documents`オプションを指定するだけです。

```yaml
schema: ./schema/*.graphql
documents: ./src/**/*.graphql
```

設定ができたら、CIで`nitrogql check`を実行しましょう。もしGraphQLコードにエラーがあれば、終了コードが0以外になります。これだけでnitrogqlを使い始めることができます。

ただし、nitrogqlの静的チェックはGraphQLの仕様に厳密に従っているということに注意してください。そのため、GraphQL Code Generatorでは問題なく動作するようなコードに対してエラーが発生することもあります。例えば、フラグメントを定義した場合、nitrogqlは他のファイルからそれを参照することを許可しません。フラグメントは同じファイルのみから参照できます。これは、（現時点では）nitrogqlがコミュニティで使われているFragment collocation技術の一部に対応していないことを意味します。私たちは現在、これらの技術をサポートする方法を調査中です。

## nitrogqlの開発

横道にそれますが、nitrogqlの開発についてもすこし紹介したいと思います。nitrogqlが動くようになるまでに、私はいくつかのものを作りました。

まず、言語処理系の基礎となるのはASTとパーサーです。私はこれらをフルスクラッチで実装しました。これはsource mapサポートの実装のために必要でした。

nitrogqlはRustで書かれています。Rustが言語処理系を書くのに向いているということを知っていたからです。そうなると、nitrogqlをnpmパッケージとして配布する方法を考えなければなりませんでした。一般的な方法は、各プラットフォーム向けに事前にビルドされたバイナリを用意することです。しかし、私は各プラットフォーム向けに複数のビルドシステムをメンテナンスしたくはありませんでした。そこで今回、nitrogqlをWebAssemblyにコンパイルし、Node.jsで実行するという方法をとることにしました。幸いなことに、Node.jsにはWASIの実験的サポートがすでにありました。WASIは、WASMモジュールがファイルシステムやその他のOS機能にアクセスできるようにするAPIです。

そう、幸いなことだと思って*いました*。しかし、Node.jsのWASIサポートは壊れていたのです。明らかに、誰もNode.jsのWASI実装を真剣に使おうと考えたことがなかったようでした。実験的な機能なので文句はありませんが、issueを報告しても、近いうちに修正されるような気配はありませんでした。そのため、nitrogqlで使うためだけに、ユーザーランドのWASI実装を作ることにしたのです。随分いろいろなものをnitrogqlのために作ったものです。

## nitrogqlの今後

nitrogqlはまだ初期段階のプロダクトです。今のところ、この記事で紹介した2つの機能が安定版としてリリースされました。これらの機能は今後も改善していきますが、まだまだやりたいことはたくさんあります。以下にいくつか紹介します。

**resolver向けの型生成**。現在のところ、nitrogqlはGraphQLオペレーションの型生成のみサポートしています。つまり、nitrogqlはGraphQLのクライアント側においてのみ役に立ちます。サーバー側においても型安全性を得るために、resolver向けの型を生成する機能を追加する予定です。

**プラグインシステム**。現在のところ、nitrogqlには少数の設定項目しかありません。追加したい機能の中には、結構opinionatedなものもあります。そのため、設定項目として追加するのではなく、プラグインシステムを追加したいと考えています。これにより、様々なユースケースに対応できるようになります。

## まとめ

nitrogqlはTypeScriptプロジェクトでGraphQLを使うためのツールチェインです。提供される機能は、コード生成と静的チェックです。もしGraphQL Code Generatorの`near-operation-file` presetが好きなら、nitrogqlのことも気にいるでしょう。nitrogqlはsource mapサポートや、さらなる安全性を提供してくれます。

さらに詳しくは、[公式ドキュメント](https://nitrogql.vercel.app/)や[GitHub](https://github.com/uhyo/nitrogql)をご覧ください。
