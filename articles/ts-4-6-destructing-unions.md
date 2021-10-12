---
title: "TypeScript 4.6で起こるタグ付きユニオンのさらなる進化"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

この記事の公開時点ではTypeScript 4.5のBetaが出たばかりといったところですが、TypeScriptのリポジトリでは早くもTypeScript 4.6をターゲットにした改善が考えられています。おそらく、大きめの新機能であるためすでにBetaが出ている4.5は避けたのでしょう。この記事ではそのうちの一つである、タグ付きユニオンに対するさらなる進化をご紹介します。PRでいうと次のものです。

https://github.com/microsoft/TypeScript/pull/46266

また、この変更によって、TypeScript 4.4, 4.5, 4.6と3連続でタグ付きユニオンが進化することになります。これらについてこの記事で紹介します。

# TypeScriptにおけるタグ付きユニオン

せっかくなので、この記事ではTypeScriptでのタグ付きユニオンについて基本的なことも解説します。タグ付きユニオンは、他にも「直和型」など色々な呼び名がありますが、英語圏のTypeScript界隈ではdiscriminated union （直訳すると「判別ユニオン / 判別共用体」）という用語がよく使われるように思われます。

これは特殊な形のユニオン型であり、ユニオン型の各構成要素であるオブジェクト型が、それらが共通して持つプロパティ（タグ）を通じてランタイムに判別できるようなものを指します。最も典型的なのは、タグとして、各構成要素に異なるリテラル型（特に文字列のリテラル型）を与えるものです。

```ts
type OkResult<T> = {
  type: "ok";
  payload: T;
};
type ErrorResult = {
  type: "error";
  payload: Error;
};
type Result<T> = OkResult<T> | ErrorResult;

function unwrapResult<T>(result: Result<T>): T {
  if (result.type === "ok") {
    return result.payload;
  } else {
    throw result.payload;
  }
}
```

上の例では、`Result<T>`型がタグ付きユニオンであり、タグとなるのは`OkResult<T>`と`ErrorResult`が共通して持つ`type`プロパティです。関数`unwrapResult`では、if文でタグをランタイムにチェックすることで、型の絞り込みを行なっています。このユニオン型の定義では、`type`プロパティの値をチェックすることで実際の値がユニオン型の構成要素の2つのオブジェクト型のうちどちらであるのか判明するようになっています。具体的には、`type`プロパティが`"ok"`であるならば`result.payload`は`T`型となります。そうでなければ、`result.payload`は`Error`型となります。

TypeScriptでは、ユニオン型の中でもこのように定義されたタグ付きユニオンには特に手厚いサポートがあり、型上で「または」を表す方法として重宝されています。

# タグ付きユニオンと型の絞り込み（TS 4.4より前）

タグ付きユニオンの強みは**型の絞り込み**に対する優れたサポートです。これまでのTypeScriptでは、このような型の絞り込みは常に**タグ付きユニオンを型に持つ変数**に対して行われるものでした。上の例でいえば、絞り込まれるのは変数`result`（`Result<T>`というタグ付きユニオン型を持つ）でした。上の例は、`result.type === "ok"`というチェックを通過した時点で`result`は`Result<T>`から`OkResult<T>`型に絞り込まれます。よって、`result.payload`は`T`型になるでした。

絞り込み条件は、`result.type === "ok"`のように、絞り込み対象の変数が明示されており、それに対するプロパティアクセスとしてタグが指定され、さらにそれに対して`===`などで比較が行われていなければいけません。これがTypeScript 4.4より前の状況でした。

# TypeScript 4.4での進化

TypeScript 4.4では、**タグの値を事前に変数に入れること**および**タグに対する判別式を変数に入れること**がサポートされ、これらの場合でも正しく型の絞り込みが行われるようになります。例えば、先ほどの例を次のように変更しても、TypeScript 4.4では動作します。

```ts
function unwrapResult<T>(result: Result<T>): T {
  const { type } = result;
  if (type === "ok") {
    // ちゃんと result が絞り込まれている
    return result.payload;
  } else {
    throw result.payload;
  }
}
```

また、次のように`===`の結果を変数に入れても正しく絞り込まれます。

```ts
function unwrapResult<T>(result: Result<T>): T {
  const isOk = result.type === "ok";
  if (isOk) {
    // ちゃんと result が絞り込まれている
    return result.payload;
  } else {
    throw result.payload;
  }
}
```

当該のPRはこちらです。

https://github.com/microsoft/TypeScript/pull/44730

# TypeScript 4.5での進化

TypeScript 4.5では、タグがテンプレートリテラル型である場合がサポートされました。やや人工的ですが次のように例を変えてみましょう。

```ts
type OkResult<T> = {
  type: `ok_${string}`;
  payload: T;
};
type ErrorResult = {
  type: `error_${string}`;
  payload: Error;
};
type Result<T> = OkResult<T> | ErrorResult;

function unwrapFooResult<T>(result: Result<T>): T {
  if (result.type === "ok_foo") {
    // ここでは result は OkResult<T> 型
    return result.payload;
  } else {
    // ここでは Result<T> 型のまま
    throw result.payload;
  }
}
```

`Result<T>`のタグが固定の文字列リテラル型ではなくテンプレートリテラル型になりました。こうすると、`OkResult<T>`のタグ（`type`プロパティ）は「`ok_`で始まる任意の文字列」という意味になり、同様に`ErrorResult`は「`error_`で始まる任意の文字列」になります。`Result<T>`の`type`プロパティとして考えられるのは、`"ok_foo"`や`"ok_pikachu"`や`"error_http"`といった文字列です。

上の例の`unwrapFooResult`関数では、その中でもタグが特定の`"ok_foo"`という値であるかどうかを確かめます。TypeScript 4.5からは、こうすることでこのif文の中（thenブランチの中）では`result`は`OkResult<T>`型に絞り込まれます。なぜなら、`"ok_foo"`は`` `ok_${string}` ``型に当てはまる値なので`result`が`OkResult<T>`である可能性がある一方で、`"ok_foo"`は`` `error_${string}` ``型に当てはまる値ではないため、`result`が`ErrorResult`である可能性が無くなるからです。

ただし、この場合、elseブランチでは`result`は`Result<T>`型のままである（`ErrorResult`に絞り込まれたりはしない）ことに注意してください。これは、`result.type`が`ok_pikachu`の場合など、`ok_foo`以外だが依然として`OkResult<T>`に属しているケースがあるからです。

この機能を追加するPRはこちらです。

https://github.com/microsoft/TypeScript/pull/46137

# TypeScript 4.6 での進化

では、いよいよTypeScript 4.6での進化を解説します。これまでは全て`result.payload`として肝心の中身にアクセスしていましたが、TypeScript 4.6では、4.4の例をさらに一歩進めて次のような形で絞り込みを行うことができます。

```ts
type OkResult<T> = {
  type: "ok";
  payload: T;
};
type ErrorResult = {
  type: "error";
  payload: Error;
};
type Result<T> = OkResult<T> | ErrorResult;

function unwrapResult<T>(result: Result<T>): T {
  // payload はここでは T | Error 型
  const { type, payload } = result;
  if (type === "ok") {
    // payload はこの中では T 型
    return payload;
  } else {
    throw payload;
  }
}
```

これまでの「あくまで型の絞り込みが行われる対象は`result`であるという」という常識を覆し、あらかじめ変数に入れておいた`payload`が絞り込まれるようになりました。このように、`type`に対するチェックを行うことで、それとは別の変数である`payload`に対して絞り込みが発生するというのがこれまでにない画期的な事象です。TypeScript 4.6より前は`payload`の方は代入された瞬間の`T | Error`固定であり、その後別の変数に対してチェックを行なっても変わることはありませんでした。

ただし、この機能が働くためには**タグが入った引数 (`type`) と中身が入った引数 (`payload`) が同じ分割代入で作られること**が必要です。つまり、次のように変えると絞り込みは行われなくなります。

```ts
function unwrapResult<T>(result: Result<T>): T {
  const { type } = result;
  const { payload } = result;
  if (type === "ok") {
    // これでは絞り込まれない！
    return payload;
  } else {
    throw payload;
  }
}
```

同様に、次のようにするのもうまくいきません。

```ts
function unwrapResult<T>(result: Result<T>): T {
  const type = result.type;
  const payload = result.payload;
  if (type === "ok") {
    // これでも絞り込まれない！
    return payload;
  } else {
    throw payload;
  }
}
```

同時に分割代入された変数でなければいけない理由は主に2つ考えられます。一つは、そのほうが変数同士の関係をトラックしやすいという実装上の問題です。もう一つは、オブジェクトの中身が変更可能であるゆえに、別々のタイミングでオブジェクトから取得されたプロパティ同士を関係させられない（あるいは、関係させて良いかどうかを追加でチェックしなければならずコストがかかる）ことが挙げられます。

ちなみに、次のように関数の引数を直に分割代入するのはOKです。むしろ、この機能の主要な需要は、このようにすることで引数オブジェクトを一度変数に入れる必要が無くなるという点にあるかもしれません。TypeScript 4.6より前では、`payload`ではなく必ず`result`が絞り込まれるという点からこれは不可能でした。

```ts
function unwrapResult<T>({ type, payload }: Result<T>): T {
  // payload はここでは T | Error 型
  if (type === "ok") {
    // payload はこの中では T 型
    return payload;
  } else {
    throw payload;
  }
}
```

このように、TypeScript 4.6の新機能は、絞り込み対象のオブジェクト（タグ付きユニオン型を持つオブジェクト）から事前に（絞り込みを行う前に）取得したプロパティが入った変数（`payload`）が、同じ分割代入に由来する別の変数（`type`）にチェックを行うことで、事後的に絞り込まれるという点に特徴があります。

当該のPRは記事冒頭でも紹介しましたが、再掲します。

https://github.com/microsoft/TypeScript/pull/46266

# TypeScript 4.6でもまだできないこと

ただし、これであらゆる場合を分割代入で解決できるようになったかと言うと、そうでもありません。先ほどまで取り扱ってきたタグ付きユニオンにはある特徴がありました。それは、`OkResult<T>`の場合も`ErrorResult`の場合も、中身を表すプロパティの名前が`payload`という共通した名前なのです。

個人的には、次のように別々の名前にするのが好みです。

```ts
type OkResult<T> = {
  type: "ok";
  value: T;
};
type ErrorResult = {
  type: "error";
  error: Error;
};
type Result<T> = OkResult<T> | ErrorResult;

function unwrapResult<T>(result: Result<T>): T {
  if (result.type === "ok") {
    return result.value;
  } else {
    throw result.error;
  }
}
```

この例では、従来`payload`だったプロパティが`OkResult<T>`では`value`という名前に、`ErrorResult`では`error`という名前になっています。上の例では従来型の方法で絞り込みを行なっているので、絞り込みの後に`result.value`や`result.error`にアクセスすることは問題なくできます。if文のthenブランチでは`result`が`OkResult<T>`に絞り込まれていて、この型には`value`プロパティが存在しているからです。

しかし、このようなタグ付きユニオンはTypeScript 4.6の恩恵を受けることができません。次のようにするとコンパイルエラーが出てしまいます。

```ts
type OkResult<T> = {
  type: "ok";
  value: T;
};
type ErrorResult = {
  type: "error";
  error: Error;
};
type Result<T> = OkResult<T> | ErrorResult;

function unwrapResult<T>(result: Result<T>): T {
  const { type, value, error } = result;
  if (type === "ok") {
    return value;
  } else {
    throw error;
  }
}
```

コンパイルエラーは次のような内容で、絞り込みの前の`result`から`value`や`error`を取得することはできないというものです。

```
error TS2339: Property 'value' does not exist on type 'Result<T>'.

12   const { type, value, error } = result;
                   ~~~~~

error TS2339: Property 'error' does not exist on type 'Result<T>'.

12   const { type, value, error } = result;
                          ~~~~~
```

つまり、`Result<T>`の段階ではこのオブジェクトに`value`や`error`が確実に存在するとは言えず、存在しないプロパティにアクセスすることは典型的なミスですからコンパイルエラーにより防がれるのです。

このことにより、その人の設計にもよりますが、TypeScript 4.6の新機能の恩恵をただちに受けられるケースはあまり多くないのではないでしょうか。

今後の展望としては2つの方向性が考えられます。一つは、TypeScript 4.6の恩恵を受けられるように、タグ付きユニオンのそれぞれの構成要素に存在するプロパティの名前を（`payload`のように）共通化するというものです。しかし、個人的にはあまりこの方向性は好みではありません。全ての場合に共通するプロパティの名前となると、それこそ`payload`のような抽象的な名前にせざるを得ず、命名があまり良くないと感じるからです。また、全ての場合に共通するプロパティならば、型の絞り込みを行う前から当該プロパティにアクセスすることができてしまいます（TypeScript 4.6の新機能ではまさにそのことを利用しているわけですが）。アクセスできてもありうる全ての場合のユニオン型になるので型安全性の問題があるわけではありませんが、本来行うべき絞り込みをし忘れる場合が想定されるので筆者はあまり好みません。

もう一つの方向性は、これは筆者の妄想なのですが、存在しないかもしれないプロパティにアクセスすることも一旦許してしまって、適切な絞り込みが行われる前に使おうとするとコンパイルエラーにするということも考えられます。

```ts
type OkResult<T> = {
  type: "ok";
  value: T;
};
type ErrorResult = {
  type: "error";
  error: Error;
};
type Result<T> = OkResult<T> | ErrorResult;

function unwrapResult<T>(result: Result<T>): T {
  // これはエラーにならない（妄想）
  const { type, value, error } = result;

  // ここで value を使おうとするとコンパイルエラーになる（妄想）
  console.log(value);
  if (type === "ok") {
    // ここで value を使うのはOK（妄想）
    return value;
  } else {
    throw error;
  }
}
```

個人的にはこの方向性になってくれると嬉しいなと思います。TypeScript 4.4から4.6までの流れによってタグ付きユニオンに対する分割代入をサポートする方向性が見出されていますから、その延長として上記のようなことも考えられます。また、すでに「適切な操作を行う前にアクセスするとエラーになる変数」という概念自体は既存のTypeScriptに存在しています。それは`let`による変数です。

```ts
let foo: number;

// ここで foo を使おうとするとエラー（未代入なので）
console.log(foo);

foo = 0;

// ここで foo を使うのはOK
console.log(foo);
```

これと同じ機構を使えば「適切な絞り込みが行われる前にアクセスするとエラーになる変数」も不可能ではないのではないかと妄想しています。このような議論は見かけたことがないので、どこかにissueにでも書いてみようかなと思っています。

# まとめ

この記事では、TypeScript 4.4から4.6で連続して起こった（あるいは起こる予定の）タグ付きユニオンのサポート強化について解説しました。タグ付きユニオンはTypeScriptで利用可能な最も強力な設計パターンの一つであり、このように機能強化が続いているのはとても嬉しいですね。