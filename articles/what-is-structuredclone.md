---
title: "structuredCloneはどんなものか"
emoji: "🏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript"]
published: true
---

**structuredClone**は、JavaScriptにおいてオブジェクトのディープコピーができる便利な関数です。

従来ディープコピーの標準化された方法が無かったため、structuredCloneの登場はJavaScriptのユーザーにとって画期的なものです。あまりに画期的であり、その便利さも分かりやすいため、出たばかりの時期はTwitterでのJavaScript豆知識ツイートの常連でした。

現在はstructuredCloneのもの珍しさは無くなり、単純に便利なAPIとして受け入れられていますが、そのせいかstructuredCloneに対する理解も単純な人が出てきているようです。

そこで、この記事ではstructuredCloneがどのようなものなのか、どうしてそのようになっているのかについて、じっくりと説明します。

# structuredCloneの歴史

筆者は自称世界一技術の歴史に興味がないエンジニアですが、structuredCloneに関してはその成り立ちがこの記事の内容に関係しているので簡単に説明します。

`structuredClone`は元々HTML仕様書で定義されたもので、2021年7月にHTML仕様にマージされました。

https://github.com/whatwg/html/pull/3414

このPRを見ると分かるように、最初の提案は2015年には上がっており、実はけっこう歴史のあるAPIです。

HTML仕様書ということはこのAPIはブラウザ環境向けに定義されたものです。しかし、昨今はDenoやNode.jsもブラウザ向けAPIを多く実装しており、structuredCloneに関してはブラウザよりも先にDenoやNode.jsがさっさと実装してしまいました。[MDN](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone)のデータによれば、Denoが2021年8月、Node.jsが2021年10月には実装を公開しており、ブラウザで最速なのは2021年11月のFirefoxです。他のブラウザは2022年以降の実装となりました。

そして、実はstructuredCloneという関数は、オブジェクトのディープコピー用のAPIとして一からデザインされたわけではありません。それよりも昔からブラウザは内部的にオブジェクトのディープコピーを行なっており、それが存在するならユーザーランドからも使われてほしいという要望からstructuredCloneが生まれました。

このブラウザの内部で行われていたディープコピーアルゴリズムがStructured Clone Algorithmであり、structuredCloneという関数名は明らかに「Structured Clone Algorithmを実行する」という意味を表したものです。ポイントは、「structuredClone関数を実装するためにStructured Clone Algorithmが定義された」のではなく、「Structured Clone Algorithmが先にあり、それを簡単に呼び出せるものとしてstructuredCloneが定義された」という順序であることです。Structured Clone Algorithm自体がいつから存在するかについては筆者は調査していません。

そのため、structuredCloneの挙動はStructured Clone Algorithmの挙動そのままです。それゆえに、汎用的なディープコピー関数として見るとやや不思議なところがあります。この記事ではこの点に注目しながらをstructuredCloneの挙動を解説します。

# structuredCloneの挙動

## SetやMapなどのオブジェクトもコピーできる

structuredCloneを使うと、プレーンなオブジェクト（ただのオブジェクトや配列）だけでなく、SetやMapといったJavaScriptに特有のオブジェクトもコピーできます。ディープコピーの方法として「`JSON.stringify`→`JSON.parse`」という方法もありますが、JSONの範囲外のオブジェクトもコピーできるのはstructuredCloneの利点です。

```js
const s = new Set([1, 2, 3]);
const s2 = structuredClone(s);

s2.has(2) // => true
s2.has(4) // => false
```

コピーできるオブジェクトはDateやRegExp、ArrayBufferなど多岐に渡ります。

## 循環したオブジェクトもコピーできる

実はstructuredCloneは循環したオブジェクトでもコピーできます。

```js
const loop = {};
loop.a = loop;

const loop2 = structuredClone(loop);
loop2.a === loop2 // => true
```

## 関数はコピーできない

一方、関数はstructuredCloneでコピーできません。（下記のエラーメッセージはGoogle Chromeのものです）

```js
const func = () => 123;

// Uncaught DOMException: Failed to execute 'structuredClone' on 'Window': () => 123 could not be cloned.
const func2 = structuredClone(func);
```

structuredCloneで関数がクローンできない理由は、もともとStructured Clone Algorithmは他の実行コンテキストにデータを移送したり、データを永続化したりする目的で定義されていたからだと思われます。例えば、他のWindowにpostMessageでオブジェクトを送信するとき、あるいはオブジェクトをIndexedDBに保存する場合にStructured Clone Algorithmが使用されます。

とくに永続化のことを考えると、structuredCloneでクローンできるためにはデータがシリアライズできる必要があります。実際の実装はともかく、structuredCloneの挙動は仕様書上では「シリアライズしてからデシリアライズする」というアルゴリズムで定義されています。

一般に、関数はシリアライズできません。そのため、関数はstructuredCloneでクローンできないのです。関数にはtoStringメソッドがあるので `() => 123` のような単純な関数であれば文字列を介してクローンできそうですが、常にそれが可能な訳ではありません。一般には、関数は環境[^note_environment]に依存します。環境をシリアライズするのは無理なので、関数をシリアライズするのは良いアイデアではありません。

[^note_environment]: 関数をとりまくスコープを表す用語です。

```js:環境に依存する関数の例
let count = 0;

const increment = ()=> ++count;
```

## 独自のクラスのインスタンスもコピーできない

structuredCloneは、自分で定義したクラスのインスタンスをコピーした場合はその情報が消えてしまいます。この場合、ただのオブジェクトとしてクローンされ、自身のPrototypeが何かという情報はコピーされません。

```ts
class MyClass {
  foo = 1;
}

const obj = new MyClass();

const obj2 = structuredClone(obj);
obj2.foo // => 1
obj2 instanceof MyClass // => false
```

独自クラスのインスタンスをコピーできない理由は、クラスも関数オブジェクトの一種でありシリアライズできないからです。

また、仮にシリアライズできたとしても困難があります。コピー先に`MyClass`が存在するとは限らないため、`MyClass`ごとコピーする必要があります。その場合、コピー先からそのインスタンスを再度元の実行コンテキストにコピーしてきた場合はどうなるでしょうか。そのオブジェクトは、元の`MyClass`のインスタンスではなく2回コピーされた別物の`MyClass`のインスタンスとなることが予想されます。

このように、SetやMapなどJavaScriptの組み込みのクラス以外については、所属クラスの同一性を保てないため所属クラスの情報をコピーするのは無理があります。

ちなみに、structuredCloneはErrorオブジェクトはコピーできますが、Errorを継承した独自クラスは当然ながらコピーできません（コピーした場合ただのErrorインスタンスになります）。

## Symbolもコピーできない

structuredCloneはプリミティブは当然コピーできます……と言いたいところですが、実はプリミティブの中でもSymbolだけはコピーできません。

```js
const sym1 = Symbol("foo");
//Uncaught DOMException: Failed to execute 'structuredClone' on 'Window': Symbol(foo) could not be cloned.
const sym2 = structuredClone(sym1);
```

データ構造的にはSymbolはコピーできそうですが、それにも関わらずSymbolをコピーできないようになっているのは、おそらくSymbolをコピーする意味がないからでしょう。

そもそもSymbolは文字列以外でオブジェクトのプロパティ名に使えることが特徴です。文字列は誰でも同じものを用意できる一方、Symbolはモジュールスコープの中などに隠しておけば他のコードから干渉されません[^note_objectgetownpropertysymbols]。偶然による事故が起きないのが利点です。

[^note_objectgetownpropertysymbols]: 唯一の例外として `Object.getOwnPropertySymbols` というAPIがありますが。

```js:Symbolの利用例
const hiddenKey = Symbol('myKey');

export function saveDataToObj(obj, value) {
  obj[hiddenKey] = value;
}

export function getDataToObj(obj) {
  return obj[hiddenKey];
}
```

そうなると、SymbolをstructuredCloneでコピーできたとしても、当然元のSymbolとは別物になるのであまり意味がないのです。

他にも、Symbolは`Symbol.for`を用いると同一のキーに対しては同一のSymbolを得ることができるというGlobal Symbol Registryの機能も備えていますが、別の実行コンテキストに送られると当然この情報も保持されません。この点も問題となるでしょう。

# Web APiのちょっとした利点

さて、あなたがNode.jsを使用する開発者だとして、Node.jsでのstructuredCloneの挙動を調べるにはどうするでしょうか。

そう、もちろん[Node.jsのドキュメント](https://nodejs.org/dist/latest-v18.x/docs/api/globals.html#structuredclonevalue-options)を見に行きますね（もっとも、このドキュメントは[MDN](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone)へのリンクが張ってあるだけですが）。

一般に、Node.jsのAPIの最も正確な説明はNode.jsのAPIドキュメントを見にいけば書いてあります。しかし、Node.jsのAPIドキュメントは（一般の多くのドキュメントと同じように）完璧にあらゆるケースを網羅している訳ではなく、細かな挙動を知りたい場合は自分で実験したり他の資料に当たったりしなければならないこともあります。

しかし、structuredCloneのようなWeb APIは、W3CやWHATWGにより管理される仕様書が存在します。JavaScript界隈の仕様書はかなり厳密に書かれており、曖昧さはほとんどなく実装による挙動の違いが入り込む余地がありません[^note_difference]。そのため、structuredCloneに関するどんな疑問もHTML仕様書を見にいけば解決します。

[^note_difference]: 明示的に実装依存と書かれている場合は別です。また、仕様書は厳密だが実装が仕様書とずれてしまっているというケースはそこそこあります。

Node.jsなどはHTML仕様書の対象外であるためHTML仕様書に厳密に従う必要はありませんが、別に仕様を乖離させる必要もないので、細かなところまで仕様書の通りに実装されていることが期待できます[^note_fetch]。

[^note_fetch]: Node.jsにはfetchが実装されていますが、これは例外となります。fetchの仕様は非常に複雑かつWebと密接に関係しているため、これに関しては細かいところまでNode.jsに実装してもあまり意味がありません。

Node.jsは結構仕様書への準拠度が高い実装をしてくれるイメージです。例えば、structuredCloneにコピーできないオブジェクトを渡した場合はNode.jsでもエラーが発生しますが、この場合に投げられるのは`DOMException`オブジェクトです。DOMというのは明らかにブラウザ用の概念ですが、Node.jsはstructuredCloneの仕様書にDOMExceptionと書いてあるので律儀にDOMExceptionを投げてくれるのです（他にも、Web由来のAPIにはだいたいDOMExceptionが使われます）。

このように、たとえNode.jsに実装されているとしても、Web由来のAPIは厳密度の高い仕様書を擁するので詳細な仕様にアクセスしやすいのが利点です。

# 結論

HTML仕様書を読もう！
