---
title: "【日本語訳】nitrogql 1.1リリース: hello type-safe resolvers!"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["graphql", "nitrogql"]
published: true
---

この記事は、昨日公開された次の記事の日本語訳です。nitrogql 1.1がリリースされresolver向けの型生成機能が使えるようになり、nitrogqlだけでサーバー側とクライアント側両方を型安全に書けるようになりました。元記事はこちらです。

https://nitrogql.vercel.app/blog/release-1.1

---

本日、nitrogql 1.1をリリースしました！

nitrogqlはTypeScriptプロジェクトでGraphQLを使うためのツールチェインです。1.1では、resolver向けの型定義の生成機能を追加しました。これにより、nitrogqlを使うことでクライアント側とサーバー側の両方で型安全にGraphQLを使うことができるようになりました。

## nitrogql 1.1の新機能

nitrogql 1.1では、1.0の機能に加えて新たに2つのTypeScriptファイルを生成できるようになりました。

- **Resolverの型定義ファイル**は、実装すべきResolverの型を定義します。
- **サーバー用GraphQLスキーマファイル**は、ランタイムにGraphQLスキーマをGraphQLサーバーに渡すのを簡単にします。

これらのファイルはGraphQLサーバーの実装に役立ちます。nitrogqlが採用している**スキーマファースト**アプローチでは、まずGraphQLスキーマを書き、それを元にクライアントとサーバーの両方を実装します。1.1のリリースにより、サーバー側のギャップが埋まりました。これでクライアント側とサーバー側の両方で型安全にGraphQLを使うことができるようになったのです！

## サーバー開発向けのnitrogql設定

これらの新しいファイルを生成するには、[設定ファイル](https://nitrogql.vercel.app/configuration/options)にいくつかのオプションを追加する必要があります。具体的には、`generate`オプションの下に`resolversOutput`と`serverGraphqlOutput`を追加します。

```yaml
schema: ./schema/*.graphql
documents: ./src/**/*.graphql
extensions:
  nitrogql:
    plugins:
      - "nitrogql:model"
    generate:
      schemaOutput: ./src/generated/schema.d.ts
      resolversOutput: ./src/generated/resolvers.d.ts
      serverGraphqlOutput: ./src/generated/server-graphql.ts
      # ...
```

この設定を追加することで、`nitrogql generate`を実行すると`resolvers.d.ts`と`server-graphql.ts`が生成されます。

## 型安全にresolverを実装する

生成された`resolvers.d.ts`は、型安全にresolverを実装するのに役立ちます。このファイルからは`Resolvers`型がエクスポートされており、これは実装すべきresolverたちのオブジェクトの型です。例えば、次のようなスキーマがあるとします。

```graphql
type Query {
  me: User!
}

type User {
  id: ID! @model
  name: String! @model
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID! @model
  title: String! @model
  content: String!
}
```

すると、生成された`Resolvers`型は次のように使うことができます。

```ts
import { Resolvers } from "./generated/resolvers";

type Context = {};

const resolvers: Resolvers<Context> = {
  Query: {
    me: async () => {
      // Returns the current user.
      return {
        id: "12345",
        name: "uhyo",
      }
    },
  },
  User: {
    email: async (user) => {
      const dbUser = await db.getUser(user.id);
      return dbUser.email;
    },
    posts: async (user) => {
      const dbPosts = await db.getPostsByUser(user.id);
      return dbPosts;
    },
  },
  Post: {
    content: async (post) => {
      const dbPost = await db.getPost(post.id);
      return dbPost.content;
    }
  },
};
```

`Resolvers`型はジェネリック型で、コンテキストオブジェクトの型を型引数として受け取ります。コンテキストはリクエストごとに作成され、すべてのresolverに渡されます。これはセッション情報やデータベース接続などをresolverに渡すために使うことができます。

## ちょっと待って、その`@model`とかいうやつは何？

そうですね。先ほどのスキーマには見慣れないものがありました。それは`@model`ディレクティブです。これはnitrogqlによって追加されたディレクティブです（より具体的には、`nitrogql:model`プラグインによって追加たものです）。このディレクティブは1.1のリリースとともに導入されました。

`@model`ディレクティブが与えられたフィールドは、その型の**モデルオブジェクト**の一部となります。これには2つの意味があります。

- `@model`ディレクティブが与えられたフィールドに対してはresolverを実装する必要がありません。[デフォルトのresolver](https://www.apollographql.com/docs/apollo-server/data/resolvers/#default-resolvers)がそれらのフィールドを処理します。
- Resolverからその型のオブジェクトを返すときには、`@model`ディレクティブが与えられたすべてのフィールドを含める必要があります。

`@model`ディレクティブはresolverを実装する際の実用性と型安全性を両立するために存在します。型安全性は、スキーマに存在するすべてのフィールドに対してresolverが実装されていることを保証することを指します。これが満たされないと、ランタイムエラーになってしまいます。しかし、すべてのフィールドに対して漏れなくresolverを実装しなければならないというのは実用性がありません。なぜなら、`id: (user) => user.id`のようなボイラープレートコードを大量に書かなければいけなくなるからです。デフォルトのresolverが役立つのがここです。デフォルトのresolverはこのような自明なresolverとして振る舞います。 

`@model`ディレクティブは、そのフィールドにデフォルトのresolverを使いたいことをnitrogqlに伝えるものです。nitrogqlはこのディレクティブを認識し、あなたが実装しなければならないresolverのリストからそのフィールドを取り除きます。重要なのは、どのresolverを実装するか、どのresolverをデフォルトのresolverに任せるかはあなた次第であるということです。だからこそ、`@model`ディレクティブも、必要なフィールドに対してあなたが手動で書く必要があるのです。どのフィールドにデフォルトのresolverを使うかをnitrogqlが自動的に判断するという選択肢もありましたが、その実装にはしませんでした。それでは柔軟性が足りないからです。

デフォルトのresolverを使うことの帰結として、resolverから返すオブジェクトには`@model`ディレクティブが与えられたすべてのフィールドを含める必要があります（これを**モデルオブジェクト**と呼びます）。これは、デフォルトのresolverはモデルオブジェクトに含まれていないフィールドを解決することができないからです。

ご存知のとおり、GraphQLのresolverは、GraphQLクエリの実行中にチェーンを形成します。つまり、resolverからオブジェクトを返すと、チェーンの次のresolverはそのオブジェクトを親オブジェクトとして受け取ります。このため、resolverが受け取る最初の引数はモデルオブジェクトになります。`@model`ディレクティブはこのようにしてresolver間のデータの受け渡しにも影響を与えます。

### `@model`の使い方

これで、なぜ`@model`ディレクティブが導入されたのかが理解できましたね。では、先ほどの例をもう一度見てみましょう。😉

スキーマを見ると、User型のモデルオブジェクトには`id`と`name`フィールドが含まれています。`email`と`posts`フィールドはモデルオブジェクトに含まれていません。同様に、Post型のモデルオブジェクトには`id`と`title`フィールドが含まれていますが、`content`フィールドは含まれていません。

```graphql
type Query {
  me: User!
}

type User {
  id: ID! @model
  name: String! @model
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID! @model
  title: String! @model
  content: String!
}
```

次に、`me` resolverの実装を見てみましょう。このresolverは`id`と`name`フィールドを含むオブジェクトを返しています。これは、User型のモデルオブジェクトにこれらのフィールドが含まれていることと合致していますね。

```ts
  Query: {
    me: async () => {
      // Userのモデルオブジェクトを返す
      return {
        id: "12345",
        name: "uhyo",
      }
    },
  },
```

`User`のresolverを見ると、`email`と`posts`のresolverが実装されています。これらのフィールドには`@model`ディレクティブが与えられていないからです。

```ts
  User: {
    email: async (user) => {
      // userは { id: string; name: string } 型
      const dbUser = await db.getUser(user.id);
      return dbUser.email;
    },
    posts: async (user) => {
      const dbPosts = await db.getPostsByUser(user.id);
      return dbPosts;
    },
  },
```

先ほども述べたように、`user`引数は`User`型のモデルオブジェクトです。そのため、`id`と`name`フィールドが含まれています。`email`のresolverは`id`フィールドを使ってデータベースからメールアドレスを取得します。

`@model`ディレクティブが与えられていないフィールドのresolverでは、追加のデータ取得が起こっていると考えることができます。`id`フィールドはデータベースからさらにデータを取得するためのキーであり、`User`型を返すresolverはモデルオブジェクトに`id`フィールドを含んでいるので、後続のresolver（`email`や`posts`など）はこれを使ってさらにデータを取得することができます。実際の状況では、[DataLoader](https://github.com/graphql/dataloader)のようなテクニックを使ってデータ取得を最適化することがあるかもしれませんが、考え方は同じです。

このことを考えると、`User`型のモデルオブジェクトに`id`フィールドが含まれているのは必然的なことです。一方で、`name`フィールドはデータ取得に使われていないので、モデルオブジェクトに含まれている必然性というのはありません。

では、なぜ`name`フィールドがモデルオブジェクトに含まれているのでしょうか？　実は、これは最適化のためです。もし`name`が頻繁に取得されるのであれば、最初のデータ取得（つまり`me` resolver）で一緒に取得しておいた方が良いでしょう。もしモデルオブジェクトに含まれていない場合、`name`を取得するためにはもう1往復のデータ取得が必要になります。`@model`ディレクティブを使うことで、型安全性を保ちつつ簡単にデータ取得を最適化することができるのです。さらに高度な最適化をしたい場合は、resolverのチェーンに入る前にクエリ全体を調べる必要がありますが、それはこんなに簡単にできることではありません。

### モデルオブジェクト全体の型を`@model`で指定する

もしもあなたが勤勉な人なら、型ごとに専用のモデルクラスを定義しているかもしれません。例えば、次のようなコードを書いているかもしれません。

```ts
class User {
  readonly id: string;
  readonly name: string;

  constructor(id: string, name: string) {
    this.id = id;
    this.name = name;
  }
  async getEmail() {
    const dbUser = await db.getUser(this.id);
    return dbUser.email;
  }
  async getPosts() {
    const dbPosts = await db.getPostsByUser(this.id);
    return dbPosts;
  }
}
```

nitrogqlはこのようなモデルの定義方法もサポートしています（あまりおすすめではありませんが）。これは、GraphQL Code Generatorの[`mappers`オプション](https://the-guild.dev/graphql/codegen/plugins/typescript/typescript-resolvers#use-your-model-types-mappers)に似ています。

このクラスをモデルオブジェクトとして使うには、`@model`ディレクティブを型そのものに大して与えます。例えば次のようになります。

```graphql
type User @model(type: "import('@/model/user').User") {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}
```

これにより、GraphQLの`User`型に対応するモデルオブジェクトは`User`クラスのインスタンスであるとnitrogqlに伝えられます。この設定の場合、resolverの実装は次のようになるでしょう。

```ts
import { Resolvers } from "./generated/resolvers";
import { User } from "@/model/user";

type Context = {};

const resolvers: Resolvers<Context> = {
  Query: {
    me: async () => {
      // Returns the current user.
      return new User("12345", "uhyo");
    },
  },
  User: {
    // `user` is an instance of User class
    id: (user) => user.id,
    name: (user) => user.name,
    email: (user) => {
      return user.getEmail();
    },
    posts: (user) => {
      return user.getPosts();
    },
  },
  Post: {
    // ...
  },
};
```

この場合、`User`の全てのフィールドに対してresolverを実装する必要があります。

## サーバー用GraphQLスキーマファイルを使用する

賢い読者の方なら、nitrogql 1.1には**サーバー用GraphQLスキーマファイル**生成機能も追加されたということを覚えているかもしれません。このファイルの役割は単純で、GraphQLスキーマを文字列としてエクスポートするだけです。例えば次のようになります。

```ts
// generated by nitrogql
export const schema = `
type Query {
  me: User!
}

// ...
`;
```

元々の`.graphql`ファイルが複数ある場合でも、それらは結合されて1つの文字列としてエクスポートされます。これにより、GraphQLサーバーを初期化する際にこれらのファイルを手動で読み込む手間が省けます。

また、このファイルは追加の安全性保証としても機能します。ランタイムで使われるスキーマが、型定義を生成する際に使ったスキーマと同じであることを保証できるためです。すべてを1つの設定ファイルにまとめるというのは、人為的なミスの可能性を減らすための素晴らしい原則です。

このファイルはGraphQLサーバーを初期化するときに利用できます。例えば、[Apollo Server](https://www.apollographql.com/docs/apollo-server/)を使う場合は次のようになります。

```ts
import { ApolloServer } from "@apollo/server";
import { schema } from "./generated/server-graphql";
import { Resolvers } from "./generated/resolvers";

const resolvers: Resolvers = { /* ... */ };

const server = new ApolloServer({
  typeDefs: schema,
  resolvers,
});
```

### スキーマのクリーンアップ

実は、サーバー用GraphQLスキーマファイルは`.graphql`ファイルを単純に結合しただけではありません。`@model`ディレクティブをすべて削除する処理がされています。

これは、ランタイムの挙動に全く影響しないディレクティブをスキーマに含めることに抵抗があるという方がいることを分かっていたからです。

私たちの考え方としては、スキーマは、ランタイムとコンパイル時の両方に通ずる*Single Source of Truth*として利用するものであるということです。コンパイル時に利用するためにGraphQLの型に何らかのアノテーションをする必要があるのであれば、スキーマに書くのが我々の好みです。

とはいえ、コンパイル時のみに利用するディレクティブをランタイムのコードから削除するのは悪いことではありません。そのため、サーバー用GraphQLスキーマファイルにはこの処理が施されています。

## `nitrogql:model`プラグイン

実は、`@model`ディレクティブは全て`nitrogql:model`という名前の組み込みプラグインによって実装されています。`@model`ディレクティブを使うには、このプラグインを有効にする必要があります。この記事の最初で少し触れているように、設定ファイルの`plugins`オプションに追加することで有効化できます。

```yaml
schema: ./schema/*.graphql
documents: ./src/**/*.graphql
extensions:
  nitrogql:
    plugins:
      - "nitrogql:model"
    # ...
```

このように、`@model`ディレクトリはオプトインの機能となっています。これは、デフォルトでカスタムディレクティブが追加されるというのはややopinionated過ぎると感じたからです。

しかし、`@model`ディレクティブなしではresolverの型定義の生成はほとんど使い物になりません。デフォルトの挙動では、各型のモデルオブジェクトにはその型のすべてのフィールドが含まれており、しかも全てのフィールドに対してresolverの実装もする必要があります。これは型安全ではありますが、実用的ではありません。

型安全性はnitrogqlにとって非常に重要なゴールです。どんなオプションの組み合わせでも型安全性は保たれるべきであり、nitrogqlは実用的よりも安全性を優先します。

型安全性を維持したままresolverの開発を実用的なものにするためには、どのフィールドをモデルオブジェクトに含むのかを開発者が指定できるようにする必要があります。これが、プラグインを通じて`@model`ディレクティブを導入した理由です。

ちなみに、プラグインがない場合のデフォルトの挙動については他の選択肢もありました。面白いかもしれないので紹介しておきます。

**全てのフィールドがモデルに含まれ、全てのフィールドに対してresolverを実装する必要がある。** これが選ばれた選択肢です。

**全てのフィールドがモデルに含まれ、resolverは全く実装する必要がない（全てのフィールドがデフォルトのresolverを使う）。** これも実は型安全です。しかし、これはGraphQL初心者を誤った方向に導く可能性があります。resolverを実装する必要がないと思わせてしまうからです。これはGraphQLの使い方としては本末転倒です。私たちは初心者にそういう使い方をするように勧めたいとは思いません。
 
**モデルのフィールドはオプショナルで、全てのフィールドに対してresolverを実装する必要がある。** これは、全てのresolverが実装されている限り必要なデータを返すことはできるので、ある意味安全です。しかし、この設定では大量のボイラープレートコードを書かなければなりません。適切な型定義があればもっと開発者体験を改善できるはずです。

**モデルのフィールドはオプショナルで、resolverもオプショナル。** これはGraphQL Code Generatorのデフォルトの挙動です。しかし、これは型安全ではないので採用できません。resolverの返り値からフィールドを省略し、かつそのフィールドに対してresolverを実装しない場合、ランタイムエラーになってしまいます。

## 次は？

実は、ロードマップには現在何もありません。これは、nitrogqlの開発が終わったということではありません。次のリリースに向けていくつかのアイデアを検討していますが、まだ決まっていないということですA

アイデアや要望があれば、[GitHub](https://github.com/uhyo/nitrogql)で教えてください。あなたのフィードバックをお待ちしています！

## まとめ

nitrogql 1.1は、クライアントとサーバーの両方で型安全性を実現するというゴールに向けて大きな一歩となりました。これで、同じGraphQLスキーマを使って両方の側で型安全性を得ることができます。これにより、GraphQLの開発がより楽しくなることを願っています。

[前回の記事](https://nitrogql.vercel.app/blog/release-1.0)では、GraphQL Code Generatorのresolverの型定義の生成はデフォルトでは型安全ではないと言いました。実際、GraphQL Code Generatorで型安全性を得られてしかも実用的な方法は、`mappers`オプションだけです。

nitrogqlは、同じようなやり方（`@model`ディレクティブを型そのものに指定する方法）をサポートしていますが、フィールドごとにディレクティブを指定する方法もあります。私たちは、この方法の方が使いやすく、resolverの実装に外部の型定義を必要としないので、こちらの方法の方が好きです。

このリリースでは、あなたにとって見慣れないものを導入することになりましたが、とても良い方向性だと我々は信じています。あなたも気に入っていただけると嬉しいです。

