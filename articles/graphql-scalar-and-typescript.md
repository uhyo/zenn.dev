---
title: "GraphQLのスカラーとTypeScriptの考察"
emoji: "🌳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["graphql", "typescript"]
published: true
---

皆さんこんにちは。筆者は最近、TypeScriptからGraphQLを使用するためのコード生成ツール**nitrogql**を開発しています（初手宣伝）。

https://nitrogql.vercel.app/

直近は、GraphQLのスカラーに関する機能拡張を行っていました。この記事では、そこで得た知見と考察について共有します。

## GraphQLのスカラーとは

GraphQLにおけるスカラーとは、それ以上分解できないデータ型のことです。GraphQLのデータはスカラー・オブジェクト・enumに分類でき、データの末端はスカラーまたはenumになります。

GraphQL仕様では組み込みのスカラー型が定義されており、以下のものがあります。

- Int
- Float
- String
- Boolean
- ID

また、カスタムスカラーとして、追加のスカラー型をスキーマ上で定義することもできます。

スカラー値がサーバー・クライアント間でやり取りされるときはシリアライズする必要があります。具体的にどのようにシリアライズするかはGraphQLの仕様の範疇ではありませんが（一応、最低限満たすべき仕様の規定はあります）、典型的にはJSONにシリアライズされます。

## コード生成ツールとスカラー

コード生成ツールでは、GraphQLのスカラー型と、それに対応するTypeScriptの型をマッピングする必要があります。例えば、GraphQLの`Int`はTypeScriptの`number`にマッピングするのが自然でしょう。

しかし、GraphQLの組み込みのスカラー型の中でも`ID`はやや特殊です。というのも、IDとしては文字列だけでなく数値も考えられるということで、入力としては文字列でも数値でも`ID`型の値として受け入れられることになっているからです。ただし、シリアライズする際は`ID`は常に文字列にするという規定があります。

つまり、シリアライズ済みの`ID`を受け取る場合は`string`型で受け取る一方で、入力としての`ID`を受け取る場合は`string | number`型で表す必要があります。

このことを表現するために、GraphQL + TypeScript向けのツールチェインでデファクトスタンダードである[GraphQL Code Generator](https://the-guild.dev/graphql/codegen)では、設定ファイルから次のようなマッピングを指定することができます。

```yaml
ID:
  input: string
  output: string | number
```

一方で、それを後追いするnitrogqlでも同様の機能を[v1.6.0でリリースした](https://nitrogql.vercel.app/blog/release-1.6)のですが、そちらでは次のように指定します。

```yaml
ID:
  send: string | number
  receive: string
```

このように、nitrogqlではinput/outputではなくsend/receiveという用語を採用しました。これはもちろんGraphQLスカラーに対する筆者の考察の結果です。ということで、この記事では、このような違いが生まれた背景について述べます。

結論を先取りすると、**サーバーとクライアントで共通の語彙を使える**というのがsend/receiveの利点です。nitrogqlの仕組み上、そうする必要があったのです。

## スカラー型の4つの使用場所

GraphQLのスカラー型からTypeScriptへの型のマッピングを考える際には、4つの異なるユースケースがあると考えられます。ここでは次のように名前を付けます。

- resolverInput
- resolverOutput
- operationInput
- operationOutput

具体例で理解しましょう。次のスキーマに対するサーバー・クライアント両方の実装を考えます。

```graphql
type Query {
  user(id: ID!): User!
}

type User {
  id: ID!
  name: String!
}
```

### resolverInput

resolverInputは、サーバー側の実装（resolver）の引数を受け取る場合です。`user`リゾルバーを次のようにTypeScriptで実装するとしましょう。

```ts
const userResolver: Resolvers<Context>["Query"]["user"] = async (
  _,
  { id }, // ← 
) => {
  // ...
}
```

この例で`id`がリゾルバーの引数として渡されています。これはスキーマ上`ID`型であり、このユースケースの場合はクライアントが数値を送ってきても文字列を送ってきても、文字列に変換済みの状態でリゾルバーに渡されます。

よって、ここでの`ID`型のマッピングは`string`とするのが正解です。

### resolverOutput

resolverOutputは、サーバー側の実装（resolver）の返り値として値が返される場合です。

```ts
const userResolver: Resolvers<Context>["Query"]["user"] = async (
  _,
  { id },
) => {
  // ...
  return {
    id: user.id, // ←
    name: user.name,
  };
}
```

このようにリゾルバーの返り値に`ID`が含まれる場合、リゾルバーは`number`を返しても`string`を返しても構いません。クライアント側に送られる前に自動的に文字列に変換されるからです。

よって、ここでの`ID`型のマッピングは`string | number`とするのが正解です。

### operationInput

残り2つはクライアント側の話です。

operationInputは、クライアント側においてオペレーションの引数として値を渡す場合です。このようなクエリを実行する場合を考えます。

```graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
  }
}
```

クライアント側のTypeScriptコードからGraphQLクエリを実行する方法は場合によりけりですが、一例としては次のようになります。

```ts
const { data } = await client.query({
  query: GetUserQuery,
  variables: {
    id: 123, // ←
  },
});
```

このユースケースにおいては、`ID`型の値として文字列でも数値でも渡すことができます。その値はそのままJSONに乗ってサーバーに送られて、サーバー側で文字列に変換されます。

よって、ここでの`ID`型のマッピングは`string | number`とするのが正解です。

### operationOutput

最後に、operationOutputは、クライアント側においてオペレーションの返り値として値を受け取る場合です。

```ts
const { data } = await client.query({
  query: GetUserQuery,
});

const id = data.user.id; // ←
```

この例での`id`は、常に文字列になります。なぜなら、サーバー側で`ID`をシリアライズした時点で文字列になり、それが送られてきているからです。

よって、ここでの`ID`型のマッピングは`string`とするのが正解です。

### 4つのユースケースのまとめ

これら4つのユースケースをまとめると次のようになります。

| ユースケース | 場所 | input/output | `ID`のマッピング |
| --- | --- | --- | --- |
| resolverInput | サーバー | input | `string` |
| resolverOutput | サーバー | output | `string ｜ number` |
| operationInput | クライアント | input | `string ｜ number` |
| operationOutput | クライアント | output | `string` |

## send/receiveモデルの導入

先ほど、GraphQL Code Generatorでは`ID`を次のように設定できると紹介しました。

```yaml
ID:
  input: string
  output: string | number
```

実は、これが当てはまるのはサーバー側だけです。クライアント側では、inputとoutputを逆転させないと正しいマッピングになりません。筆者はこの点を問題視したためGraphQL Code Generatorと同じセマンティクスは採用できないと考えました。

そこで、nitrogqlでは**send/receive**という分け方を採用しています。

| ユースケース | 場所 | send/receive | input/output | `ID`のマッピング |
| --- | --- | --- | --- | --- |
| resolverInput | サーバー | **receive** | input | `string` |
| resolverOutput | サーバー | **send** | output | `string ｜ number` |
| operationInput | クライアント | **send** | input | `string ｜ number` |
| operationOutput | クライアント | **receive** | output | `string` |

こうすれば、次の設定はサーバー/クライアント問わずに正しいマッピングを与えます。

```yaml
ID:
  send: string | number
  receive: string
```

名前の意図としては、**send**は「これから相手に送られる値」であり、**receive**は「相手から受け取った値」を意図しています。特に、**receive**の値は通信を経た値であり、クライアントとresolverに介在するGraphQLサーバーによって型の変換（`ID`の場合は数値から文字列への変換）が行われたあとの値です。そのため、**receive**においては変換後の型にマッピングする必要があります。

では、input/outputというモデルを採用したGraphQL Code Generatorは間違いなのでしょうか。筆者の考えでは、必ずしもそうではないと考えています。GraphQL Code Generatorとnitrogqlでは仕組みが違うため、前者はinput/outputでも大丈夫だったが、後者はそれではだめだったということです。

というのも、GraphQL Code Generatorではクライアント側の型の生成とサーバー側の型の生成は別々のプラグインが行なうのであって、スカラー型のマッピングもプラグインのオプションとして指定できるからです。つまり、サーバー側のプラグインのinput/outputとクライアント側のプラグインのinput/outputを逆に指定することによって、正しいマッピングを与えることができます。

一方で、nitrogqlは1つの設定がサーバー側とクライアント側の両方に影響するため、input/outputではなくsend/receiveという普遍的に通用するモデルを採用する必要がありました。nitrogqlでは、このsend/receiveのモデルが`ID`型に最も合致していると考え採用しました。

ちなみに、nitrogqlでは4つのユースケースの型を全部別々に指定する方式もサポートしています。

## 他のカスタムスカラーはどうなのか

以上のような仕組みで、`ID`の場合はなんとかなりました。

しかし、他のカスタムスカラーで同じ議論が通用するとは限りません。特に、スカラーの値がTypeScript上ではオブジェクトで表現されるような場合が怪しいですね。

そこで、便利なカスタムスカラーを集めたライブラリである[GraphQL Scalars](https://the-guild.dev/graphql/scalars/docs)から`DateTime`型を取り上げてみましょう。`DateTime`型の実装はGitHubで見ることができます。

https://github.com/Urigo/graphql-scalars/blob/3f0aa62abc48073cf398d6a4514ef84b345b904e/src/scalars/iso-date/DateTime.ts

JavaScriptにおけるカスタムスカラーの実装は`GraphQLScalarType`のインスタンスのことであり、スキーマ上でカスタムスカラーを定義することに加えて、これをGraphQLサーバーに読み込ませることでカスタムスカラーが使用可能になります。`GraphQLScalarType`インスタンスは、値のシリアライズとデシリアライズを担当します。

よって、値が何にシリアライズされ、何にデシリアライズされるのかが分かれば、TypeScriptにおけるマッピングを決めることができますね。

ということで`DateTime`の実装を見てみると、次のようになっています。

| メソッド | 入力 | 出力 |
| --- | --- | --- |
| `serialize` | `Date ｜ string ｜ number` | `Date` |
| `parseValue` | `Date ｜ string` | `Date` |

つまり、resolverは`Date | string | number`を返すことができて、それは`Date`にシリアライズされます。一方で、サーバーへの入力は`Date | string`が受け付けられます。

しかし、`Date`にシリアライズされるということはどういうことでしょうか。JSONで`Date`オブジェクトを送ることはできません。そのため、JSONにシリアライズされて送られるときに`Date`はさらに文字列に変換されます。ここの変換は、`Date`の`toJSON`メソッドによって行われます（つまり、GraphQLサーバーの範疇ではなく、JavaScriptの仕様によるものです）。

つまり、resolverから返された`DateTime`型のデータは次のような一生を辿ります。

| resolverの返り値 | シリアライズ後 | 通信経路（JSON） | クライアント |
| --- | --- | --- | --- |
| `Date ｜ string ｜ number` | `Date` | `string` | `string` |

resolverが何を返しても、クライアントには文字列で届きます。また、クライアント側では、`string`を`Date`に戻すような処理は行われません。なぜなら、`GraphQLScalarType`の実装はサーバー側の話であって、クライアント側が関知するものではないからです。通常、クライアント側はスキーマを知らないため、送られてきた文字列がスキーマ上`DateTime`であるということをランタイムに判断できません。

また、クライアントからサーバーに`DateTime`を送るときもJSONを経由することを意識する必要があります。

以上のことから、`DateTime`において最も適切なマッピングは次のようになります。

```yaml
DateTime:
  resolverInput: Date
  resolverOutput: Date | string | number
  operationInput: Date | string # 本来はstringだが、DateもtoJSONでstringになるので
  operationOutput: string
```

ぐちゃぐちゃですね。send/receiveとは何だったのでしょうか。

このように、JSONにシリアライズできない値を扱おうとしたらsend/receiveモデルは破綻します。そのため、nitrogqlでは上記のように4パターンを別々に指定する必要があります。input/outputはサーバー側とクライアント側で別々に指定すれば何とか大丈夫ですが、それも実質4パターン指定していますね。

ちなみに、記事公開時点では、GraphQL Code Generator + GraphQL Scalarsの組み合わせでは全パターン統一して`Date | string`にマップされるのがデフォルトの挙動になります。改善の余地ありですね。

個人的には、カスタムスカラーがクライアント側で結局JSONを逸脱できないのが微妙で、あまり使い勝手が良くないなと感じています。また、コード生成ツールの設定を細かく頑張らないと、実態と異なる型になってしまいます。nitrogqlでこの状況を何とか改善できると嬉しいですね。

## まとめ

カスタムスカラーとかいうやつ、皆本当にちゃんと使いこなしてるのかこれ？

完