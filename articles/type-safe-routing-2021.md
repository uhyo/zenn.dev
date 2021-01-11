---
title: "2021年の密かなトレンド？　“型安全ルーティング”の概観"
emoji: "🚥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "react"]
published: true
---

2020年は、**型安全ルーティング**が密かに盛り上がりを見せた年でした。この記事では、TypeScript周りのエコシステムで発生した型安全ルーティングという概念とこれまでの流れを振り返ってご紹介します。

## ルーティングとは

この記事でいう**ルーティング**は、URL（特に`/user/uhyo`といったパス部分）を見てコンテンツを出し分ける機構のことを指します。ルーティングは、主にSPA (Single Page Application) で必要となります。SPAはどのようなURLでも同じHTMLとJavaScriptが動作し、JavaScriptによってアドレスに対応したコンテンツが表示されます。まさに、ルーティングがSPAの根幹となっています。また、一般のウェブサーバーも、異なるURLに対するリクエストには異なるレスポンスを返しますから、ここでもルーティングが行われていることになります。

従来は、**文字列ベースのルーティング**が行われてきました。例えば、Node.js向けのwebサーバーライブラリの[Express](https://expressjs.com/)ではこのようなコードでルーティングを行います。

```ts
// / にアクセスされたときの処理
app.get("/", (req, res) => {
  res.send("Top Page");
});
// /hello にアクセスされたときの処理
app.get("/hello", (req, res) => {
  res.send("Hello, world!");
});
// :userId は任意の文字列を受け入れる
app.get("/user/:userId", (req, res) => {
  const userId = req.params.userId;
  res.send(`User is ${userId}`);
});
```

このコード例では、`app.get`の第1引数によりルートを文字列で指定し、第2引数の関数でそのルートにアクセスされたときの処理を記述します。`:userId`のような特別な記法によって任意の文字列を受け入れることも可能です。例えば、`/user/:userId`は`/user/pikachu`や`/user/uhyo`といったパスを受け入れます。`:userId`に実際に何が入っていたかは`req.params.userId`で取得できます。

もう一つSPAの例も見てみましょう。React向けのルーターライブラリとしてデファクトスタンダードの地位を獲得している[react-router](https://reactrouter.com/)の例です。

```tsx
<Switch>
  <Route path="/" exact>
    <p>Top Page</p>
  </Route>
  <Route path="/hello" exact>
    <p>Hello, world!</p>
  </Route>
  <Route path="/user/:userId" exact>
    <UserPage />
  </Route>
</Switch>
```

こちらでも、パスは文字列で指定されていることが分かります。`:userId`に何が入ったのかについては、`UserPage`コンポーネントの中で次のように`useParams`フックを使うことで取得できます。

```tsx
const UserPage = () => {
  const params = useParams<{ userId: string }>();

  return <p>User is {params.userId}</p>;
};
```

### 文字列ベースのルーティングの危険性

以上のような文字列ベースのルーティングは、これまで**型安全性**が欠如していました。特に、上の例で出てきた`:userId`が問題となります。例えば、Expressの場合、デフォルトの型定義（`@types/express`のもの）では`req.params`が`any`型となっています。当然、`req.params.userId`も`any`型です。

![Expressのreq.paramsの型](https://storage.googleapis.com/zenn-user-upload/b3j6q6rxvw39evuyw9ah42hzjnqk)

ここに明らかな危険性があります。`:userId`と`req.params.userId`の両方に出てくる`userId`というのは当然一致していなければいけません。ルーティング文字列中の`:userId`をExpressが見て`req.params.userId`に対応する文字列を入れてくれるからです。しかし、TypeScriptの型定義の上ではそれは明らかではありません。`req.params`はどんなルーティング文字列に対しても対応しなければいけないからです。これにより、例えば間違って`req.params.user`としてしまうというミスをしたとしても、TypeScriptのコンパイルエラーは発生しません。これが文字列ベースのルーティングの危険性の一例です。

react-routerの場合も同様の危険性があります。次のコード（再掲）では、`useParams`に`{ userId: string }`という型引数を渡しています。これは「`userId`というパラメータがあるからよろしく」と`useParams`に教えていることになります。

```tsx
const UserPage = () => {
  const params = useParams<{ userId: string }>();

  return <p>User is {params.pikachu}</p>;
};
```

ここでの問題は`useParams`を騙し放題だということです。例えば次のようにすれば`params.pikachu`が（実際には存在しないのに）使えるでしょう。この例では、実際には`params.pikachu`は存在しない（`undefined`である）のに対し、`params.pikachu`は`string`型を得ることになり、やはり型安全性が壊れてしまっています。

```tsx
const UserPage = () => {
  const params = useParams<{ userId: string; pikachu: string }>();

  return <p>User is {params.pikachu}</p>;
};
```

明らかに、これを型安全になるように修正するのは困難です。なぜなら、`UserPage`は独立したコンポーネントであるため、その外側（`<Route path="/user/:userId">`）の情報を得ることができないからです。

以上のように、現在主流の文字列ベースのルーティングは型システムとの相性が悪く、型安全性を高めることができないという問題がありました。より具体的に言えば、各ルートにおいて得られる情報のスキーマが`/user/:userId`のような文字列で定義されていることにより、それを型システムに組み込むことが困難だったのです。さらに、react-routerの例に見えるように、そもそもスキーマの情報があったとしてもそれを活かせるようなAPIになっていませんでした。ここではreact-routerを例に出していますが、Vueなど他のライブラリでも状況は変わりませんでした。

### ナビゲーションの危険性

これに付随する問題として、**ナビゲーション**の型安全性にも改善の余地がありました。ナビゲーションとは、SPAにおいて別のURLに遷移させることを指します。例えばreact-routerでは次のようなコードでナビゲーションを行います。

```ts
history.push("/user/uhyo");
```

ここに潜む改善の余地とは、**存在しないパスを指定しても型エラーが起きない**ということです。例えば、次のような打ち間違いがあったとしても実際に動かしてみるまで間違いに気づくことができません。

```ts
// userをusrにtypoしてしまった
history.push("/usr/uhyo");
```

「こんな所まで型エラーにこだわる必要があるのか？」という疑問を抱いた方もいるかもしれませんが、複雑な遷移フローや要件変更に付随するURLの変更などを経験してみてください。きっと考えが変わるでしょう。

## 型安全ルーティングに対する3つのアプローチ

型安全ルーティングという概念はこれまでほとんど気にも留められていませんでしたが、2020年はその状況に変化が起こりました。型安全性ルーティングに対する3つの異なるアプローチが登場したのです。

### 最初の転機：TypeScript 4.1

2020年9月、TypeScript界隈がにわかにお祭り騒ぎとなりました。それはTypeScript 4.1の新機能として**Template Literal Types**が発表されたからです。詳しい解説などは他の記事に譲りますが、この機能によりTypeScriptの型レベル文字列処理機能が大きく向上しました。template literal typesにより実装が可能になったものとして、[型レベル四則演算](https://dqn.fish/articles/2020-12-13-type-level-calculator)や[型レベルJSONパーサー](https://twitter.com/__gfx__/status/1330465518990614529?s=20)、[型レベルパーサーコンビネータ](https://zenn.dev/todesking/articles/d5c88b046c59f2ed76bb)といった作品が挙げられます。

Template literal typesを用いれば、`"/user/:userId"`のような文字列を解析してこのパスが`userId`というパラメータを持つという判定を型レベルで行うことが可能です。そのようなアイデアはtemplate literal typesの登場直後から複数観測されています。ここで型安全ルーティングという概念が初めて日の目を見ることになりました（それ以前にもほとんど気づかれないような規模の試みが1つか2つあったことも分かっていますが）。具体的な実装例として例えば次のようなものが挙げられます。

https://github.com/menduz/typed-url-params

このライブラリでは、`ParseUrlParams<"/user/:userId">`という型が`{ userId: string }`と計算されます。これにより、例えばExpressの`req.params`の型を改善することができるでしょう。

```ts
app.get("/user/:userId", (req, res) => {
  // req.params が { userId: string } となっていて型安全！
  const userId = req.params.userId;
});
```

### 文字列ベースからの脱却というアプローチ

Template literal types登場の少し前、2020年8月に筆者は**Rocon**をリリースしました。これはまさに、React製SPAにおいて型安全なルーティングを実現するためのライブラリです。Roconのリリース時には一定の評価とともに、そこまでルーティングの型安全性にこだわることへの懐疑の目があったと記憶しています。

https://github.com/uhyo/rocon

このライブラリの特徴は、**文字列によるルーティング定義をやめてコードによる定義にした**という点にあります。Template literal typesの登場以前だったこともあり、`"/user/:userId"`のような文字列から型安全性に繋がる情報を引き出すことは全く不可能だったため、この記事で述べたような安全性の問題を解決するためには文字列を捨てなければいけないことは必然でした。また、template literal typesが登場した今となっても、文字列によるルート定義をsource of the truthとするアプローチは表現力に乏しいため（あとで詳説します）、Roconのような非文字列ベースのスキーマを基とするアプローチの方が表現力の面で有利です。

のちに、2020年12月には同じく非文字列ベースのルーティング機能を持つ`@fleur/froute`が登場しました。こちらはRoconと同様な非文字列ベースルーティングの恩恵を得つつ、template literal typesのメリットも取り込んでいるのが特徴です。また、どちらかというとRoconはCSRを重視したAPIを、`@fleur/froute`はSSRを重視したAPIを提供しています。

https://github.com/fleur-js/froute

たとえばRoconの例では、`"/user/:userId"`のような文字列を使わずに、同等のルートを次のように定義します。

```ts
// ここで /user を定義
const toplevelRoutes = Rocon.Path().route("user");

// ここで /user/:userId を定義
const userRoutes = toplevelRoutes._.user.attach(
  Rocon.Path()
).any("userId", {
  action: ({ userId }) => <UserPage userId={userId} />
});
```

文字列ひとつのお手軽さに比べるとかなり仰々しいのが難点ですが、`/user`とか`/user/:userId`と行った概念が全て文字列ではなくオブジェクトで表現される点にAPIの特徴があります。これの比較対象となるのはreact-routerのこの例です。

```tsx
// react-routerの例
<Route path="/user/:userId" exact>
  <UserPage />
</Route>
```

react-routerでは`userId`を`UserPage`コンポーネントの中で取得していたのに対して、Roconでは`UserPage`の外から`userId`が与えられています。Roconではこの`userId`は`action`のコールバック関数の引数から与えられています。現在のURLから`userId`を取り出すところはRoconがやってくれて、それをこのような非文字列のコードにすることで型安全性を確保しています。

また、ナビゲーション時はこのようにします。

```ts
// react-routerの場合
history.push("/user/uhyo");
// Roconの場合
navigate(userRoutes.anyRoute, { userId: "uhyo "});
```

一見してみると、template literal typesが登場した今となってはただAPIが仰々しいだけではないかと思われるかもしれませんが、それ以外の利点もあります。これについても詳しくは後述しますが、特に重要なものは2つです。一つは、取り扱えるデータソースが幅広いという点、またもう一つは上の例にあるようにナビゲーションの型安全性もカバーしているという点です。

### コード生成によるアプローチ

ここまでルーティングの型安全性に対する2つのアプローチを紹介しましたが、実はNext.jsのようなフレームワークのユーザーはこれらの恩恵を全く受けることができませんでした。なぜなら、Next.jsは文字列ベースともオブジェクトベースとも異なる、**ファイルシステムベースのルーティング**を採用していたからです。

ファイルシステムベースのルーティングでは、ディレクトリ構造がそのままパスの構造となります。例えば、Next.jsでは`pages/user/[userId].tsx`という場所にファイルを作ることで、そのファイルがエクスポートするコンポーネントが自動的に`/user/:userId`に相当するパスを担当することになります（`[userId].tsx`は文字通り、角カッコを含んだファイル名とします）。

ファイルシステム文字列ベースのルーティングならば型システムに組み込まれた文字列という概念が相手なのでまだマシでしたが、ファイルシステムが相手となるとTypeScriptの型システムでは手も足も出ません。前節で紹介したライブラリのうち、RoconはNext.jsをサポートしておらず、`@fleur/froute`は一応API上はNext.jsをサポートしているものの、型安全ルーティングではありません。

このような状況で有効なのが**コード生成**によるアプローチで、実際にそれを実装したのが**pathpida**です。これはコード生成による型安全性の提供を得意とするaspidaファミリーの一部です。

https://github.com/aspida/pathpida

Pathpidaのクライアントを走らせておくことによって、ファイルシステム（`pages`以下のディレクトリ構造）をウォッチしてその情報を含んだTypeScriptコードを生成してくれます。このコードはナビゲーション時に使用します。

```tsx
// pathpida無しの場合
<Link href="/user/uhyo">...</Link>
// pathpidaありの場合
<Link href={pagesPath.user._userId("uhyo").$url()}>...</Link>
```

このように、pathpidaを使うことでURLを指定する際に動的な部分（`[userId]`の部分）のみを受け取ってURL文字列を返すようなAPIが提供されます。これにより、`/user`を`/usr`と打ち間違えるようなミスは型レベルで回避されます。

Next.jsなどのフレームワークでは、ルーターライブラリ（react-routerのような）はNext.js本体と一体化しています。ファイルシステムベースルーティングはこのNext.js本体（と一体化したルーター）の機能です。そのため、Next.jsのルーター部分を別の3rdパーティーライブラリで置換するのが難しく、Next.jsユーザーの型安全性のためにはコード生成に頼らなければいけない状況となっています。

## 3つのアプローチの比較と色々な安全性

ここまで紹介した3つのアプローチは、一見して「型安全ルーティング」という共通の目標を持っているように見えますが、実は各々が目標としているところは多少異なっています。特に、型安全ルーティングは実は**データを受け取る部分の型安全性**（`/user/:userId`にアクセスされたときに`:userId`の部分に入ったデータを受け取る部分が型安全であること）と**ナビゲーション部分の型安全性**（`/user/:userId`にアクセスしたいときにそのURLを作る部分が型安全であること）の2つに大別されます。そのほかにも、Next.js対応などまで考えると、それぞれのアプローチの特色が見えていきます。

3つのアプローチの特徴・対応領域を表にまとめると次のようになります。

|  | データ受け取りの型安全性 | ナビゲーションの型安全性 | データソースの広さ | ファイルシステムベース対応 |
| - | :-: | :-: | :-: | :-: |
| template literal types | ○ | × | × | × |
| オブジェクトベース（Rocon, froute） | ○ | ○ | ○ | × |
| コード生成（pathpida） | × | ○ | △ | ○ |

これまではそれぞれのアプローチ（表の行方向）で見てきたので、今度は安全性の分類（表の列方向）を見ていきましょう。

### データ受け取りの型安全性

- template literal types: ○
- Rocon/froute: ○
- pathpida: ×

この記事で**データ受け取りの型安全性**と読んでいるのは、`/user/:userId`のようなパスを担当するプログラムが、`:userId`に何が入っているのか型安全に取得できることを指します。例えば`userId`の型が`any`などではなく正しく`string`となっていて、`usrId`のようにtypoしたときにエラーが出るならば型安全です。

例えばtemplate literal typesでは次のようにこの安全性を達成できました。

```ts
app.get("/user/:userId", (req, res) => {
  // req.params が { userId: string } となっていて型安全！
  const userId = req.params.userId;
});
```

Roconの例を再掲すると、Roconの場合は型推論により下のコードの`action`に渡される引数が`{ userId: string }`型となっているので、これも型安全です。

```tsx
const userRoutes = toplevelRoutes._.user.attach(
  Rocon.Path()
).any("userId", {
  // actionの引数として { userId: string } が渡される
  action: ({ userId }) => <UserPage userId={userId} />
});
```

コード生成（pathpida）は×となっていますが、pathpidaに現状そのような機能が存在しないので×としました。しかし、原理的には可能と思われます。最も、その場合生成するファイルが1つでは無理なのでなかなか厄介なのですが。

実際、Next.jsのでは例えば次のように`userId`を取得します（`getServerSideProps`を使用する場合）。

```tsx
type ServerSideProps = {
  // ...
};
type Query = {
  userId: string
}

export const getServerSideProps: GetServerSideProps<ServerSideProps, Query> = async ({ context }) => {
  // string | undefined 型
  const userId = context.params?.userId;
  return {
    props: {
      // ...
    }
  }
};
```

このように、`:userId`部分は`context.params.userId`に入っています（ただし、`context.params`は`undefinde`の可能性があります）。本来ならば`context.params`の中身はファイル名（`pages/user/[userId].tsx`）によって決まりますが、上のコード例では`Query`型によって「`context.params`の中身は`userId: string`ですよ」と`GetServerSideProps`に教えています。この`Query`型の定義を間違えてしまう可能性があるため、これは型安全ではありません。

Pathpidaには現状ではこの部分（データ受け取りの型安全性）に対するサポートがありません。

### データソースの広さについて

- template literal types: ×
- Rocon/froute: ○
- pathpida: △

先ほどの表に「**データソースの広さ**」とありましたが、この突然出てきた新出概念は何を指しているのでしょうか。これは、データをどこから受け取ることができるかを指しています。

実は、これまでこの記事ではデータソースとして1種類のみを取り扱ってきました。それはパス名です。つまり、`/user/uhyo`のようなパス名を`/user/:userId`のようなルート定義にマッチさせて`:userId`に相当する`uhyo`を取り出す処理について、これまで議論してきたわけです。

このパス名というデータソース以外に、あと2つメジャーなデータソースがあります。それは、**クエリパラメータ**と**history state**です。クエリパラメータは、`/user/uhyo?page=2`のようなURLの`page=2`の部分です。また、history stateはHTML5 History APIの概念であり、URL上には現れないもののhistory entry（ブラウザの履歴の1単位）に紐づいたデータを保存することができます。特にSPAでは、これら3種類のデータソースを適材適所で使い分ける必要があります。

そして、データソースの広さというのは、これら3種類のデータソースをどれだけサポートしているかという指標です。

Template literal typesはあくまで`"/user/:userId"`のような文字列を相手にするものなので、クエリパラメータやhistory stateのサポートはありません（頑張って文字列のスキーマを拡張すれば作れるかもしれませんが、そのような実装は今のところ見たことがありません）。よって×としています。

Roconは3種類全てに対応しているので○です。

Pathpidaはクエリパラメータにのみ対応しており、history stateのサポートが無いため△としています。

### ナビゲーションの型安全性

- template literal types: ×
- Rocon/froute: ○
- pathpida: ○

**ナビゲーションの型安全性**は、遷移したい先のURLを指定する部分を型安全に書くことができるかという観点です。まず、型安全でない例としてreact-routerの例を見てみます。

```tsx
// 手続き的な例
history.push("/user/uhyo");
// 宣言的な例
<Link to="/user/uhyo">りんく</Link>
```

どちらもただの文字列であり、`/usr/uhyo`のように打ち間違えても型エラーが起きないので型安全ではありません。

Template literal typesはデータ受け取りの型安全性に特化したアプローチであり、ナビゲーションについては何のサポートも無いので×です。文字列ベースのアプローチでは、ナビゲーションをサポートするのは原理的に不可能でしょう。

Roconなどのオブジェクトベースのアプローチはナビゲーションの型安全性をサポートしているので○です。オブジェクトベースのアプローチは、データ受け取りの型安全性とナビゲーションの型安全性を両立するための最も自然な方法です。Roconの例はこんな感じです。

```tsx
// 手続き的な例
navigate(userRoute, { userId: "uhyo" });
// 宣言的な例
<Link route={userRoute} match={{ userId: "uhyo" }}>りんく</Link>
```

Pathpidaは、template literal typesとは逆にナビゲーションの型安全性に特化したアプローチなので当然○です。

```tsx
// 手続き的な例
router.push(pagesPath.user._userId("uhyo"));
// 宣言的な例
<Link href={pagesPath.user._userId("uhyo")}>りんく</Link>
```

なお、多くのアプローチではクエリパラメータもサポートしていますが、ライブラリによって取り扱いにも多少の差異があります。例えば、Roconはパスの一部とクエリパラメータを一緒くたに扱いますが、frouteやpathpidaは両者を区別して扱います。

### ファイルシステムベース対応

- template literal types: ×
- Rocon/froute: ×
- pathpida: ○

先ほども述べたように、Next.jsのようなファイルシステムベースのルーティングシステムに太刀打ちできるのがpathpidaの最大の特徴です。他のアプローチはNext.js環境下では有効ではありません。

## まとめ

この記事では2020年後半から盛り上がりを見せた**型安全ルーティング**の現状をまとめました。型安全ルーティングのための3つのアプローチを紹介し、ルーティング型安全性という性質のより詳細な分類と合わせて解説しました。

まだ型安全ルーティングについて詳しくなかったという方は、ぜひこの機会に型安全ルーティングの導入について検討してみましょう。型安全ルーティングを実践したいという方、この記事を参考にして自分にあったライブラリを探してみましょう。最後に、ライブラリの比較表を再掲しておきます。

|  | データ受け取りの型安全性 | ナビゲーションの型安全性 | データソースの広さ | ファイルシステムベース対応 |
| - | :-: | :-: | :-: | :-: |
| template literal types | ○ | × | × | × |
| オブジェクトベース（Rocon, froute） | ○ | ○ | ○ | × |
| コード生成（pathpida） | × | ○ | △ | ○ |