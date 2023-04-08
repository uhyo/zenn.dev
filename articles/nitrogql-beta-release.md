---
title: "緊急解説！　突如出現したnitrogqlの中身と裏側"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "graphql", "nitrogql"]
published: true
---

皆さんこんにちは。これは、筆者が最近公開した[nitrogql](https://github.com/uhyo/nitrogql)を宣伝する記事です。nitrogqlの概要や、開発にあたっての裏話などを紹介します。

https://github.com/uhyo/nitrogql

# nitrogqlとは

**nitrogql**は、TypeScriptコードからGraphQLを使用するためのツールです。有体にいえば、[graphql-code-generator](https://the-guild.dev/graphql/codegen)を置き換えることを目指して開発しています。具体的には、**`.graphql`ファイルからTypeScriptの型定義を生成する**機能を備えています。

例えば、次のようなクエリがあったとします。

```graphql:myTodoList.graphql
query ($unfinishedOnly: Boolean) {
  todos(filter: { unfinishedOnly: $unfinishedOnly }) {
    id
    body
    createdAt
    finishedAt
    tags {
      id
      label
      color
    }
  }
}
```

これに対しては、次のような型定義が生成されます。

```ts:myTodoList.d.graphql.ts
import type { TypedDocumentNode } from "@graphql-typed-document-node/core";
import type * as Schema from "../../generated/schema";

type QueryResult = Schema.__SelectionSet<Schema.Query, {
  todos: (Schema.__SelectionSet<Schema.Todo, {
    id: Schema.ID;
    body: Schema.String;
    createdAt: Schema.Date;
    finishedAt: Schema.Date | null;
    tags: (Schema.__SelectionSet<Schema.Tag, {
      id: Schema.ID;
      label: Schema.String;
      color: Schema.String;
    }, {}>)[];
  }, {}>)[];
}, {}>;

type QueryVariables = {
  unfinishedOnly: Schema.Boolean | null;
};

declare const Query: TypedDocumentNode<QueryResult, QueryVariables>;

export { Query as default };


//# sourceMappingURL=myTodoList.d.graphql.ts.map
```

そして、この型定義を使って、TypeScriptコードからGraphQLクエリを実行することができます。次はReactと[urql](https://formidable.com/open-source/urql/)を使った例です。

```tsx:myTodoList.tsx
import { useQuery } from "urql";
import Query from "./myTodoList.graphql";

const MyTodoList = () => {
  const [{ data }] = useQuery({ query: Query, variables: { unfinishedOnly: true } });
  return (
    <ul>
      {data?.todos.map((todo) => (
        <li key={todo.id}>
          <div>{todo.body}</div>
          <div>{todo.createdAt}</div>
          <div>{todo.finishedAt}</div>
          <ul>
            {todo.tags.map((tag) => (
              <li key={tag.id}>
                <div>{tag.label}</div>
                <div>{tag.color}</div>
              </li>
            ))}
          </ul>
        </li>
      ))}
    </ul>
  );
};
```

お分かりのように、nitrogqlでは`.graphql`ファイルをTypeScriptコードから直接読み込むやり方を推奨しています。その理由は筆者の好みによるところが大きいですが、TypeScript内のGraphQLクエリのようなものをサポートするとパーサーの実装が複雑になるといった理由もあります。

普通の実行環境では`.graphql`ファイルを直接importすることはできないので、nitrogqlではJavaScriptに変換するwebpack loaderとRollup pluginを提供しています。これらがあれば今どきの多くのフロントエンド環境に対応できます。これらは同等の機能を提供する既存のライブラリもありますが、nitrogqlから提供することで、生成された型定義とランタイムが一致することを保証できます。

# nitrogqlの特徴

上記のように、nitrogqlのひとつの特徴は`.graphql`ファイルを直接読み込むことを推奨していることです。これは、TypeScript 5.0で`allowArbitraryExtensions`というコンパイラオプションが追加されたことが関係しています。このオプションは、その名が示す通り、真の意味で任意の拡張子のimportを許可するものです。今後は色々な拡張子のファイルをそのままimportするのがブームになりそうだと思ったのでこの方向でいくことにしました。

他の特徴として、上記の生成コードの最後の行を見ると分かる通り、nitrogqlはsource mapの生成をサポートしています。筆者は既存のプロジェクトでgraphql-code-generatorを使っていましたが、source mapの生成機能が無いことを特に不満に思っていました。当初はgraphql-code-generatorを修正してsource mapの生成機能を足そうかとも思いましたが、実装を見た感じ無理そうでした。これが、新たにnitrogqlを作ることにした理由の一つです。

source mapがあることの利点は、エディタの定義ジャンプ機能を使う際に、生成された型定義ではなく元の`.graphql`ファイルにジャンプできることです。定義を見たときにより分かりやすいほか、定義のほうの修正が必要な場合に素早く修正できるなど、開発効率上のメリットがあります。

![VS CodeのPeek Definition機能でスキーマを表示しているところ](/images/nitrogql-beta-release/screenshot-peek-schema.png)

# nitrogqlの機能

nitrogqlの機能として、上述の型定義の生成機能のほかに、**GraphQLの静的チェック**も備えています。つまり、GraphQLで定義されたスキーマとオペレーションを解析し、実行時にバリデーションエラーが発生するであろうところをあらかじめ指摘する機能です。存在しない型名を指定しているところ、存在しないフィールドを指定しているところ、型の不一致などを指摘してくれます。

この機能を実装した問題意識としては、一部の静的エラーはgraphql-code-generatorも指摘してくれるものの、完璧ではないことがありました。例えば、GraphQLではフィールドに引数を指定することができますが、この引数の型がスキーマとオペレーションで食い違っていてもgraphql-code-generatorを指摘してくれず、実際に実行してランタイムエラーを貰うまで気づくことができません。例えば:

```graphql
type Query {
  todos(filter: TodoFilter): [Todo!]!
  #             ^ 型ではTodoFilter型を受け取ることになっているが……
}
```

```graphql
query ($unfinishedOnly: Boolean) {
  todos(filter: $unfinishedOnly) {
    #           ^ 間違ってBoolean型を渡しても怒られない！
    id
    body
  }
}
```

nitrogqlでは、可能な限りあらゆるエラーを静的に指摘します。このためにGraphQLの仕様を読み込んでかなり詳しくなりました。

以上の2点が現在nitrogqlで提供されている主な機能であるとともに、既存ツールであるgraphql-code-generatorとの違いです。

# nitrogqlの今後

次にやりたいのは、resolverの型定義の生成です。現在のところ、nitrogqlが生成してくれる型定義はクライアント向け（GraphQLクライアントと一緒に使うことが想定されている型定義）です。これに加えて、GraphQLサーバーの実装を書くときに役立つ型定義も提供したいと考えています。

ちなみに、nitrogqlでは**schema-first**なアプローチを推しています（対義語はcode-firstです）。つまり、APIのsource of truthとしてまず書かれるのがGraphQLのスキーマであり、サーバー側とクライアント側の両方をそのスキーマに沿って実装するということです。個人的に実装よりもインターフェースが正として扱われるのが好きなのと、技術スタックを乗り換えやすいという利点に魅力を感じているためです。

以前に筆者が書いたこちらの記事を覚えているでしょうか。こちらの記事でも、実際にschema-firstなアプローチが採られているプロダクトにおけるresolverの型定義の問題とその解決策を紹介しました。

https://zenn.dev/babel/articles/graphql-typing-for-babel

とくに、記事の後半では「カスタムディレクティブによる解決」として、`@customResolver`というディレティブを用意してそれを見て生成されるresolverの型定義が調整されるという方法を紹介していますが、この実装のためにgraphql-code-generatorのプラグインをカスタマイズしていました。

しかし、あまりきれいな実装とは言えず、graphql-code-generatorをベースにしているときれいな実装が不可能なように思われました。そこで、このような解決策をよりきれいでメンテナンスしやすい形で実装する土台を作ることも、nitrogqlの目標になっています。

個人的には、nitrogqlを通じて、GraphQLスキーマを色々とデコレートするやり方を広めていきたいです。上の記事に対する反応として、クライアントに見せるインターフェースに内部実装の情報が含まれているのは良くないという意見がありましたが、それは本体のスキーマからクライアントに見せる用のスキーマを出力する機構を作れば解決しそうです。

しかしこのようなやり方はさすがに万人受けはしなそうなので、プラグインシステム的なものを作れないかと構想しています。graphql-code-generatorから得られた教訓が活かせるはずです。

# nitrogqlのこだわり

筆者はTypeScriptに一家言あるタイプの人間なので、nitrogqlが生成する型もこだわりを持って作られています。ここではその一端を紹介します。

## ユニオン・オプショナルの取り扱い

まず、GraphQLの型システムにはユニオン型の概念がありますから、これをTypeScriptのユニオン型に対応させるのは当然です。GraphQLでは、ユニオン型が返ってくるフィールドでは`__typename`メタフィールドを見ることで具体的な型が何なのか調べることができます。これはTypeScriptではユニオン型の絞り込みに対応しています。GraphQLではfragment spread（`... F`）により特定の型の場合のみフィールドを取得できる構文もありますから、それも型に反映しなければいけません。

とはいえ、それくらいはTypeScriptでGraphQLを扱うには必須級であり、既存ツール（graphql-code-generator）でも対応しています。nitrogqlに特有の工夫としては、`@skip`と`@include`の扱いがあります。これらは引数に与えられた`Boolean!`がそれぞれ`false`・`true`の場合のみフィールドを取得するという意味のディレクティブです。逆に言えば、引数によってはフィールドが取得されない可能性があるということです。そのため、普通にやるとこれらのディレクティブが付けられたフィールドはオプショナルなものとして扱う必要があります。

では、次のケースではどうなるでしょうか。

```graphql
foo @skip(if: $flag)
bar @include(if: $flag)
```

この場合、`$flag`の値によって`foo`と`bar`のどちらか一方が取得されることになります。両方が取得される・どちらも取得されないということはありえません。nitrogqlは、このことを反映した型を生成することができます。この例に対しては、次のような型が生成されます。

```ts
| {
    foo?: never;
    bar: string;
  }
| {
    foo: string;
    bar?: never;
  }
```

他のツールでは`foo`と`bar`をそれぞれオプショナルにして終わるので、nitrogqlのほうがより厳密な型を生成できています。

理想を言えば、この`$flag`というのはクエリに与えられた引数に由来しているため、具体的な値を型引数として受け取ってそれを用いるのがよいのですが、`TypedDocumentNode`の定義上（あとTypeScriptでHKT直接を扱えない関係で）このような型は表現できないのでひとまず諦めています。

## フィールドのエイリアス性の扱い

他にもこだわりが反映されている点として、オペレーション（queryやmutationの総称です）の型に出現するフィールドは、大元のスキーマにおけるフィールドのエイリアスとして扱われるように注意して型定義が作られています。その帰結として、VS Codeにおいてqueryから得られたフィールドに対してPeek Referencesを行うと、スキーマの定義が表示されたり他のqueryで使われている箇所が表示されたりするようになっています。

![queryから得られたフィールドにPeek Referencesを行なっている様子のスクリーンショット](/images/nitrogql-beta-release/screenshot-aliased-field.png)

ここまでこだわった型定義はあまりありません。実装としては、homomorphic mapped typeを使うと元のオブジェクトとエイリアス性のある別のオブジェクト型が作れることを利用しています。ただ、GraphQLの構文としてフィールドに別名を付けられるものがあり、これが使われている場合はエイリアス性を保つことができません。筆者よりTypeScriptに詳しくてこれを直せる方がいたら、ぜひ助言をお願いします。

# nitrogqlの技術的に面白いところ

nitrogqlはもちろんこれまでのツールを凌駕する開発体験を提供するために作られていますが、技術的に面白いところもいくつかあります。面白くて、筆者のGitHubの草が久しぶりに生い茂りました。

![GitHubの草。今年2月〜現在（4月）は土日も含めて毎日コミットがあることが分かる](/images/nitrogql-beta-release/github-weeds.png)

まず、実装言語はRustです。ここ何年か流行りの、JavaScript向けツールチェインをRustとかGoで書いてみましたの流れに乗っています。nitrogqlはある種の言語処理系ですから、Rustを使うととても快適に書けます。

JavaScript向けのツールをRustで書いた場合、バイナリをどのように配布するかが問題になります。やはりnpmでインストールできるようにしたいですね。典型的なのはバイナリの実行可能形式を配布する方法です。この方法は、各環境向けのクロスコンパイルが必要だったり、[各環境向けのパッケージを用意するのが大変だったり](https://logmi.jp/tech/articles/325858)といった点で面倒です。

ということで、今回はWASMにコンパイルして、Node.jsのWASI実装を通じて実行する方式を採用しました。これであれば、環境によらずに1つのWASMバイナリをパッケージに同梱するだけで済み、簡単です。ネイティブコンパイルに比べて速度で劣る可能性がありますが、V8がかなり頑張っていて十分速いので問題なさそうです。

ただ、nitrogqlのようなCLIツールをNode.jsのWASIで提供するという試みはおそらくこれまで行われていません。その理由は、筆者がやってみたところ、[（恐らく）ARM Mac上では`fd_readdir`がうまく動かない](https://github.com/nodejs/node/issues/47193)というバグを発見したからです。このバグの影響を受けた場合、ディレクトリのファイル一覧を取得する操作が無限ループに陥ってしまいます。ファイルシステムを扱うCLIをこの状況で動かすのは現実的ではなく、これまでに報告もされていなかったことから、多分誰もやっていないのだろうと思われます。

尤も、Node.jsのWASIはexperimentalという位置付けですから、使われていなくても無理はありません。とはいえ、使ってあげないと実験になりませんから、今回は果敢に使っています。前述のバグについてはまだ直っていませんので、[問題のある関数を自前の実装で置き換える](https://github.com/uhyo/nitrogql/blob/1985cf0b373e225d4539588a257fdd9788e1d5a7/packages/cli/bin/shim.mjs)ことで回避しています。

# まとめ

この記事では、筆者が最近公開したnitrogqlについて解説しました。記事公開時点ではbeta版という扱いですが、条件が合えばすぐに試せますので、気になる方はぜひ挑戦してみてください。今回は[GitHub](https://github.com/uhyo/nitrogql)のほかに公式ドキュメントも用意しています。細かい使い方はそちらに載せています。

https://nitrogql.vercel.app/

ちなみに、炎に足が生えたようなnitrogqlのロゴはAIに描いてもらいました。リクエストは「GraphQL logo on fire」です。