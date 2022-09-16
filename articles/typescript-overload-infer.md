---
title: "オーバーロードされた関数型から引数の型や返り値の型を取り出す方法"
emoji: "🎳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

TypeScriptでは、型レベル計算を用いて関数型から引数の型や返り値の型を取り出すことができます。この操作を簡単に行うための`Parameters`や`ReturnType`という組み込み型も用意されています。

しかし、オーバーロードされた関数型の場合は普通のやり方が通用しません。そこで、この記事ではオーバーロードされた関数型から引数の型や返り値の型を取り出す方法を解説します。

# 復習: 関数型から引数の型や返り値の型を取り出す方法

普通の関数型の場合、Conditional Typesと`infer`を用いることで、パターンマッチの要領で関数型からその一部分を抜き出すことができます。

```ts
type Func = (left: number, right: number) => string;

// [left: number, right: number] 型
type Params = Func extends (...args: infer Params) => any ? Params : never;
// string 型
type Ret = Func extends (...args: any[]) => infer Ret ? Ret : never;
```

これらのパターンはよく使うわりに長いので、TypeScriptの標準ライブラリに組み込み型として用意されています。

```ts
type Func = (left: number, right: number) => string;

// [left: number, right: number] 型
type Params = Parameters<Func>;
// string 型
type Ret = ReturnType<Func>;
```

# 復習: オーバーロードされた関数型

TypeScriptでは、オーバーロードされた関数型というものが存在します。関数のオーバーロードは古くからTypeScriptに存在する機能であり、この機能を使って宣言された関数がオーバーロードされた関数型になります。

```ts
function niceFunc(arg: number): string;
function niceFunc(arg: string): number;
function niceFunc(arg: number | string): string | number {
  // ...
}

// ↓これがオーバーロードされた関数型
type Func = typeof niceFunc;
```

上の例で宣言された関数`niceFunc`は、「`number`型の引数を渡されれば`string`を返し、`string`型の引数を渡されれば`number`型を返す」という意味の関数宣言を持っています。TypeScriptの関数オーバーロードは、このように本体となるfunction宣言の前に、型だけのシグネチャを並べるという構文を持ちます。

他にも、function宣言を介さずにオーバーロードされた関数型を直接宣言する方法としては、オブジェクト型の中に複数のコールシグネチャを並べる方法があります。

```ts
// 上のコードのFunc型と同じ
type Func = {
  (arg: number): string;
  (arg: string): number;
};
```

さらに、複数の関数型をインターセクション型で合成した場合も同じようにオーバーロードされた関数型が得られます。

```ts
type F1 = (arg: number) => string;
type F2 = (arg: string) => number;

// これも上のコードのFunc型と同じ
type Func = F1 & F2;
```

# 本題: オーバーロードされた関数型から引数の型や返り値の型を取り出したい

ここからが本題です。上記のような方法で定義されたオーバーロードされた関数型に対して、`Parameters`や`ReturnType`を使うとどうなるでしょうか。試してみましょう。

```ts
type Func = {
  (arg: number): string;
  (arg: string): number;
};

// [arg: string] 型
type Params = Parameters<Func>;
// number 型
type Ret = ReturnType<Func>;
```

このように、`Parameters`や`ReturnType`を使った場合、**一番最後のシグネチャの中身が結果として得られます**。逆に言えば、**一番最後以外の型は取り出せない**のです。記事の冒頭で述べた「普通のやり方が通用しない」とはこのことです。

実はオーバーロードされた関数型に対してConditional Typesと`infer`を用いると、一番最後のシグネチャ以外は無かったことにされてしまうのです。これがこの記事で取り扱う問題です。

:::details オーバーロードされた関数型とConditional Typesの関係について詳しく

オーバーロードされた関数型に対してConditional Typesを使用したときの振る舞いは、実は`infer`を用いるかどうかによって変化します。

次の例では、`infer`を使っていないためオーバーロードされた関数型のすべてのシグネチャが認識されているような挙動をします。

```ts
type Func = {
  (arg: number): string;
  (arg: string): number;
};

type T1 = Func extends (arg: number) => string ? true : false; // true
type T2 = Func extends (arg: string) => number ? true : false; // true

type T3 = Func extends (arg: number) => number ? true : false; // false
type T4 = Func extends (arg: string) => string ? true : false; // false
```

一方で、`extends`の右辺（の関数型部分）に`infer`が含まれる場合、最後のシグネチャのみが認識されるようになります。

```ts
type Func = {
  (arg: number): string;
  (arg: string): number;
};

type T1 = Func extends (arg: infer X) => string ? X : false; // false
type T2 = Func extends (arg: infer X) => number ? X : false; // string

type T3 = Func extends (arg: number) => infer _ ? true : false; // false
type T4 = Func extends (arg: string) => infer _ ? true : false; // true
```

このように、オーバーロードされた関数型と`infer`の相性が悪いようです。
:::

# 結論: オーバーロードされた関数型から引数の型や返り値の型を取り出す方法

普通に関数型と`infer`を使っても、オーバーロードされた関数型から取り出せるのは最後のシグネチャの内容だけであり、それ以外の部分は取り出せませんでした。

この問題を回避するには、**Conditional Typesの右辺にもオーバーロードされた関数型を用いてパターンマッチさせます**。

コードで表すと、こういうことです。

```ts
type Func = {
  (arg: number): string;
  (arg: string): number;
};

type Sig = Func extends {
  (...args: infer Params): infer Ret;
  (...args: any[]): any;
}
  ? { params: Params; ret: Ret }
  : never;
```

こうすると、次の結果が得られます。

```ts
type Sig = {
  params: [arg: number];
  ret: string;
}
```

最後のコールシグネチャではなく、1番目のコールシグネチャの内容が取り出せていることが分かります。

このように、**同じ数のオーバーロードを持つ関数型**をConditional Typeの右辺に配置し、欲しい箇所に`infer`を配置することで好きな場所の情報を得ることができます。

もし左辺のオーバーロードが3つなら、右辺も3つにします。

```ts
type Func = {
  (arg: boolean): boolean;
  (arg: number): string;
  (arg: string): number;
};

// type Sig = {
//   params: [arg: boolean];
//   ret: boolean;
// }
type Sig = Func extends {
  (...args: infer Params): infer Ret;
  (...args: any[]): any;
  (...args: any[]): any;
}
  ? { params: Params; ret: Ret }
  : never;
```

:::details 両辺にオーバーロードがある場合の詳細な挙動

オーバーロードされたシグネチャは、下から順番にマッチされるようです。例えば、次のコードでは右辺の`infer`が下から2番目にあるので、下から2番目の情報が取り出されます。

```ts
type Func = {
  (arg: boolean): boolean;
  (arg: number): string;
  (arg: string): number;
};

// type Sig = {
//   params: [arg: number];
//   ret: string;
// }
type Sig = Func extends {
  (...args: infer Params): infer Ret;
  (...args: any[]): any;
}
  ? { params: Params; ret: Ret }
  : never;
```

また、`infer`による情報の取得は、他のシグネチャの内容に関係なく、位置だけを見て行われるようです。次の例では`(args: number): string`を先に消費したと思いきや、`infer`は位置だけ見ているのでやはり`(args: number): string`がマッチするという結果になっています。

```ts
type Func = {
  (arg: number): string;
  (arg: string): number;
};

// type Sig = {
//     params: [arg: number];
//     ret: string;
// }
type Sig = Func extends {
  (...args: infer Params): infer Ret;
  (args: number): string;
}
  ? { params: Params; ret: Ret }
  : never;
```

実際の実装を確認していないので推測になりますが、`extends`の右辺にあるオーバーロードされたシグネチャはそれぞれ独立にチェックされ、相互作用しないものと思われます。

---

右辺のほうがオーバーロード数が多い場合の処理は、`infer`がどの位置にあるかによって変わります。

次の例では右辺のほうがオーバーロード数が多いですが、マッチには成功します。これは、`infer`がある「下から2番目」は左辺にも存在しているからです。

```ts
type Func = {
  (arg: number): string;
  (arg: string): number;
};

type Sig = Func extends {
  (args: any): any;
  (...args: infer Params): infer Ret;
  (args: any): any;
}
  ? { params: Params; ret: Ret }
  : never;
```

一方で、次のように左辺に存在しない位置（下から3番目）に`infer`があった場合はマッチに失敗します。

```ts
type Func = {
  (arg: number): string;
  (arg: string): number;
};

// type Sig = never
type Sig = Func extends {
  (...args: infer Params): infer Ret;
  (args: any): any;
  (args: any): any;
}
  ? { params: Params; ret: Ret }
  : never;
```

このように、`infer`を含むシグネチャは位置のみを見てマッチ対象を選択します。これは、次のようにシグネチャの一部のみが`infer`だったとしても同じです。

```ts
type Func = {
  (arg: number): string;
  (arg: string): number;
};

// type Sig = never
type Sig = Func extends {
  (args: any): any;
  (arg: number): infer Ret;
}
  ? { ret: Ret }
  : never;
```

この場合、右辺の一番下のシグネチャに`infer`があるため、`(arg: number): infer Ret`のマッチ対象は左辺の一番下の`(arg: string): number`になります。これは引数が違うのでマッチ失敗となり、`Sig`は`never`になってしまいます。このように、位置以外は何も見ていないようです。

ただし、`infer`を含まないシグネチャの場合は位置は関係なく部分型関係を確認してくれます。`infer`が絡んだ瞬間に位置しか見なくなると理解しましょう。

```ts
type Func = {
  (arg: number): string;
  (arg: string): number;
};

// type Sig = true
type Sig = Func extends {
  (arg: string): number;
  (arg: number): string;
}
  ? true
  : false;
```

ちなみに、普通の関数型はオーバーロードされたシグネチャが1つだけの関数型と見なされます。記事の冒頭で説明した「関数型を`extends`の右辺に使うと最後のオーバーロードの結果が取得される」という挙動もこれで説明できます。

:::

まとめると、**オーバーロードされた関数型の特定の位置の情報を抜き出したい場合、同じ形のオーバーロードされた関数型をConditional Typesの右辺に配置すべし**となります。

# 残る問題

お察しの通り、以上の方法には問題があります。それは、そもそも左辺の型の構造（オーバーロードの数や順番）を知っていないと欲しい情報が取り出せないということです。

理想的には、オーバーロードされたコールシグネチャを分解して取り扱いやすい形に変換できたらいいのですが、筆者の知る限りその方法はありません。現在のところ、この記事の例のように場所を指定して`infer`で取り出すのが精一杯のようです。もしやるのであれば、ひと昔の型定義によく見られたように、10個くらいまで全部列挙して力技でやる必要がありそうです。

# まとめ

この記事では、オーバーロードされた関数型から、最後のものだけではなく好きな位置のシグネチャの引数の型や返り値の型を取り出す方法を紹介しました。

今のところ、これがオーバーロードされた関数型の中身に干渉する唯一の方法のようです。より取り回しのいい方法は見つかっていません。
