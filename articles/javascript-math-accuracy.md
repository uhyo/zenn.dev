---
title: "JavaScriptの数値計算はどれくらい正確なのか"
emoji: "🧮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "ecmascript"]
published: true
---

JavaScriptは様々な用途で使われるプログラミング言語で、色々な用途に対応するための一通りの機能が揃っています。その中には、**数値計算**の機能も含まれています。

数値計算、特に小数の計算においては、計算結果の**正確性**が度々問題になります。プログラムにおいては、色々な要因で計算結果には誤差が発生します。一例として、浮動小数点数の場合は数を表現するために使えるビット数が有限であることから、計算結果は真の値（数学的な意味での正しい計算結果）と異なる値になることがあります（いわゆる丸め誤差）。例えば、JavaScriptの数値はIEEE 754 倍精度浮動小数点数（いわゆるdouble）ですが、doubleでは`1 / 10`の結果（0.1）を正確に表すことができず、結果の浮動小数点数は（10進数で書き下すと）`0.100000000000000005551115123125782702118158340454101562`となり完璧に0.1ではありません。最も、これはdoubleで表現可能な限界値であり、doubleの世界ではこれ以上に正確に0.1を表すことができないのですから仕方がありません。

さて、JavaScriptというプログラミング言語では、さまざまな数値計算の結果はどれくらい正確なのでしょうか。この記事では、JavaScriptの言語仕様を定義する文書である[ECMAScript仕様書](https://tc39.es/ecma262/)を紐解いて、JavaScriptの数値計算においてどれくらいの正確性が保証されているのかを調べます。

## 四則演算

まずは基本的な計算である四則演算について調べてみましょう。JavaScriptで`a + b`という式がどのように評価されるのかは[12.8.3.1 Runtime Semantics: Evaluation](https://tc39.es/ecma262/#sec-addition-operator-plus-runtime-semantics-evaluation)で定義されています。ここは文字列の`+`による結合なども含まれているので数値を扱う部分を探して進んでいくと、[6.1.6.1.7 Number::add](https://tc39.es/ecma262/#sec-numeric-types-number-add)にたどり着きます。

NaNやInfinity, 0などの扱いを除いた通常のケースを調べると次のように書かれています（以降すべて日本語訳は筆者によるもの）。

> In the remaining cases, where neither an infinity, nor a zero, nor NaN is involved, and the operands have the same sign or have different magnitudes, the sum is computed and rounded to the nearest representable value using [IEEE 754-2019](https://tc39.es/ecma262/#sec-bibliography) roundTiesToEven mode. If the magnitude is too large to represent, the operation overflows and the result is then an infinity of appropriate sign. The ECMAScript language requires support of gradual underflow as defined by [IEEE 754-2019](https://tc39.es/ecma262/#sec-bibliography).

> （訳）残りのケースでは、どちらのオペランドもInfintyやゼロやNaNではなく、両オペランドの符号が同じか、または異なる絶対値を持っています。この場合、両者の和が計算され、[IEEE 754-2019](https://tc39.es/ecma262/#sec-bibliography)最近接偶数への丸めによって（IEEE 754 倍精度浮動小数点数で）表現可能な値に丸められます。もし絶対値が大きすぎて表現不可能な場合は、オーバーフローが発生して結果は適切な符号のInfinityとなります。ECMAScriptでは、[IEEE 754-2019](https://tc39.es/ecma262/#sec-bibliography)で定められるgradual underflowのサポートが要求されます。

ということで、足し算はdoubleで表現可能な範囲で最も正確な結果が得られるようです。

引き算の場合（[6.1.6.1.8 Number::subtract](https://tc39.es/ecma262/#sec-numeric-types-number-subtract)）は、`x - y`の結果は`x + (-y)`の結果に等しくなるとされています。`(-y)`の計算（[6.1.6.1.1 Number::unaryMinus](https://tc39.es/ecma262/#sec-numeric-types-number-unaryMinus)）は符号を変えるだけなので誤差は発生しませんね。

掛け算（[6.1.6.1.4 Number::multiply](https://tc39.es/ecma262/#sec-numeric-types-number-multiply)）・割り算（[6.1.6.1.5 Number::divide](https://tc39.es/ecma262/#sec-numeric-types-number-divide)・剰余（[6.1.6.1.6 Number::remainder](https://tc39.es/ecma262/#sec-numeric-types-number-remainder)）の場合も、計算結果を最近接偶数への丸めによって表現可能な値に丸められると定義されています。

これらの四則演算はすべてIEEE 754-2019で定められた通りの結果になるとされています。

## 冪乗

冪乗は`x ** y`で表現され、`x`の`y`乗を表す演算です。

[6.1.6.1.3 Number::exponentiate](https://tc39.es/ecma262/#sec-numeric-types-number-exponentiate)から引用します。

> It returns an implementation-approximated value representing the result of raising base to the power exponent, subject to the following requirements:
>
> （訳）`base`を`exponent`した値を表すimplementation-approximatedな値を返します。ただし、次の条件に従う必要があります：

これまでと毛色が異なり、[implementation-approximated](https://tc39.es/ecma262/#implementation-approximated)な値という概念が出てきました。なお、次の条件という部分は0やInfinityやNaNといった特殊ケースな値に関する定義がされています。

implementation-approximatedについては次のように定義されています。

> An implementation-approximated facility is one that defers its definition to an external source while recommending an ideal behaviour. While conforming implementations are free to choose any behaviour within the constraints put forth by this specification, they are encouraged to strive to approximate the ideal. Some mathematical operations, such as Math.exp, are implementation-approximated.
>
> （訳）implementation-definedという機構は、理想的な挙動を推奨しつつ、具体的な定義を外部に委譲するものです。仕様で定められた制約を満たしていれば処理系がどのような挙動をしても仕様に適合しますが、処理系は理想的な結果を近似するように努力することが勧められています。`Math.exp`のようないくつかの数学的計算はimplementation-approximatedとして定義されています。

ざっくりとまとめると、**なるべく正確な結果を出すように心がけるべきだが、仕様として結果の正確性は保証しない**ということです。

冪乗の場合、仕様で定められた制約とは仕様書に記載されている0・Infinity・NaNなどに関する挙動を指しており、通常の結果については「累乗の計算結果を表すimplementation-approximatedな値」としか定義されていません。

つまり、極端な話、「**仮数部の計算が面倒臭いから`3 ** 2`は`8`**」という処理系があったとしてもECMAScript仕様違反ではないのです。現実にはそんな極端な処理系は無いとは思いますが、現実的な話としても、最後の1ビットまで正確な結果が出ることは期待しないほうがよいでしょう。

## `Math`系の演算

JavaScriptでは、`Math`オブジェクトを通して様々な数学的な演算機能が標準で提供されています。これらの制度がどうなっているのか見てみましょう。

### `Math.abs`

与えられた数の絶対値を返す関数です（[20.3.2.1 Math.abs](https://tc39.es/ecma262/#sec-math.abs)）。符号しか扱わないので結果は正確です。

### `Math.acos`・`Math.acosh`・`Math.asin`・`Math.asinh`・`Math.atan`・`Math.atanh`・`Math.atan2`

逆三角関数たちです。`Math.atan2`は特によくお世話になりますね。これらの結果は全て**implementation-approximated**です。

ただし、`Math.acosh`・`Math.asinh`・`Math.atanh`に関しては、入力の絶対値が1より大きい場合は返り値が`NaN`になると定められています。定義域くらいは気にしているわけですね。また、結果が0になる場合（`Math.acos(1)`、`Math.asin(0)`, `Math.atan(0)`など）は具体的に定められています。これは、結果が0の場合は+0と-0の2種類があるためと考えられます。例えば、`Math.asin(-0)`は`0`ではなく`-0`となります。

つまり、「**中途半端な角度は面倒くさいから全部$\frac{\pi}{4}$を返す**」みたいな実装でも仕様違反ではないということです。

### `Math.cbrt`・`Math.sqrt`

3乗根・2乗根です。こちらも、結果が0・NaN・Infinityとなる場合をのぞいてimplementation-approximatedです。

つまり、「**小数の計算は大変だから`Math.sqrt(5)`は2**」のような処理系があっても仕様準拠となります。

### `Math.ceil`・`Math.floor`・`Math.round`・`Math.trunc`

小数を整数に変換する関数たちです。`ceil`は切り上げ、`floor`は切り捨て、`Math.round`は四捨五入、`Math.trunc`は0に近い方向への丸めとなります。

例えば`Math.ceil`の仕様から引用すると（[20.3.2.10 Math.ceil](https://tc39.es/ecma262/#sec-math.ceil)）、入力`n`に対して次のように定義されています。つまり、（JavaScriptの数値で表現可能な範囲で）最も正確な値が返るということです。安心ですね。他の3種も同様です。

> Return the smallest (closest to -∞) integral Number value that is not less than n.

### `Math.clz32`

与えられた数値を符号なし32ビット整数として扱い、leading zero bitsの数を数える計算です。ビット演算なので結果は正確です。

### `Math.cos`・`Math.cosh`・`Math.sin`・`Math.sinh`・`Math.tan`・`Math.tanh`

三角関数たちです。逆三角関数の場合と同様に、結果は全てimplementation-approximatedな値です。

つまり、「**`Math.cos`の結果とかどうせ0から1の間だし全部0.5を返そう**」というような処理系でも仕様準拠となります。また、`Math.cos`や`Math.sin`は結果が[-1, 1]の間におさまることという定義もされていませんので、`Math.cos`が唐突に**10**を返したりしても仕様違反にはならないでしょう。

### `Math.exp`・`Math.expm1`・`Math.pow`

`Math.exp(x)`は$e^x$を返す関数で、`Math.expm1(x)`は$e^x-1$を返す関数です（$e$は自然対数の底）。`Math.pow(x, y)`は$x^y$です。結果はいずれもimplementation-approximatedです。`Math.expm1`については次のように書かれています（[20.3.2.14 Math.exp](https://tc39.es/ecma262/#sec-math.exp)）。

> The result is computed in a way that is accurate even when the value of x is close to 0.
>
> （訳） 結果はxが0に近い時でも正確に計算されます。

つまり、`Math.expm1(x)`は`Math.exp(x) - 1`とするよりも正確な結果が期待できます。ただし、結局どちらもimplementation-approximatedなので仕様上の保証は何もないのですが。

### `Math.fround`

与えられた数値をfloat （IEEE-754 単精度浮動小数点数）に変換します。このときの変換は最近接偶数への丸めによって行うと定義されており、表現可能な最も正確な結果となります。JavaScriptにfloat型という型はないのでこの値は再度floatからdoubleに変換されて得られますが、このときに誤差は発生しません。

### `Math.hypot`

いくつかの引数$x_1, \dots, x_n$を受け取り、$\sqrt{x_1^2 + \dots + x_n^2}$を返す関数です。

仕様には次のように書かれています。

> Implementations should take care to avoid the loss of precision from overflows and underflows that are prone to occur in naive implementations when this function is called with two or more arguments.
>
> （訳）処理系は、ナイーブな実装において起こりがちな精度低下やオーバーフロー・アンダーフローを避けるように注意すべきです。

つまり、自分で`Math.sqrt(x1 ** 2 + ... + xn ** 2)`という実装をするよりも、`Math.hypot`を使ったほうが計算結果の誤差が少ないことが期待できます。

とはいっても、結果は案の定implementation-approximatedなので仕様上は無保証なのですが。

### `Math.imul`

与えられた2つの引数を符号なし32ビット整数として扱い、両者の積の下位32ビットを符号あり32ビット整数として解釈した整数を返します。整数演算なので結果は正確です。

言葉で解説すると何をしたい関数なのか分かりにくいですが、[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/imul)を見ると次のように書いてあります。

> The `Math.imul()` function returns the result of the C-like 32-bit multiplication of the two parameters.

つまり、この挙動はC言語などにおける32ビット整数の乗算の挙動を模倣しているのです。

### `Math.log`・`Math.log1p`・`Math.log10`・`Math.log2`

対数計算系の関数です。`Math.log`は自然対数で、`Math.log1p`は与えられた値に1を足した数の自然対数を返します。`Math.log1p`は`Math.exp1m`と似た境遇にありますね。

やはり全て結果はimplementation-approximatedなので結果は無保証です。

「**自然対数の計算は面倒くさいから`Math.log(8)`の結果は2を底として計算して3でいいや**」という実装があったとしても仕様準拠となります。

### `Math.max`・`Math.min`

与えられた引数の中で最大・最小の物を返す関数です。浮動小数点数の計算ではないので誤差はありません。

### `Math.random()`

数値計算ではありませんが、0以上1以下のランダムな数値を返す関数です。

仕様（[20.3.2.27 Math.random](https://tc39.es/ecma262/#sec-math.random)）から定義を引用します。

> Returns a Number value with positive sign, greater than or equal to +0𝔽 but strictly less than 1𝔽, chosen randomly or pseudo randomly with approximately uniform distribution over that range, using an implementation-defined algorithm or strategy. This function takes no arguments.
>
> （訳）implementation-definedなアルゴリズムまたは戦略を用いて乱数的にまたは擬似乱数的に一様分布から選ばれた、0以上1未満の数値を返します。この関数は引数を取りません。

、乱数生成に使用するアルゴリズムは[implementation-defined](https://tc39.es/ecma262/#implementation-defined)とされています。つまり、処理系は好きな方法で乱数を生成してよいということです。

また、生成された乱数の性質に関する定義は特にありません。

つまり、乱数と言いつつ常に0しか返さない`Math.random`も仕様に準拠しています……と言いたいところですが、実はそうでもありません。仕様にはもう一文あり、次の制約が課されています。

> Each Math.random function created for distinct realms must produce a distinct sequence of values from successive calls.
>
> （訳）異なるrealmにおいて作られたそれぞれの`Math.random`関数は、それぞれ異なる乱数列を生成しなければいけません。

ここで出てきたrealmという用語は、簡単に言えば実行環境のことだと思ってください。ブラウザで例を出すと、iframeの中と外は異なる実行環境となります。

つまり、`Math.random`は毎回異なる乱数列を生成しないといけないということになります。常に0しか返さない`Math.random`はこれに反していますので、仕様準拠にはなりません。少し安心ですね。

### `Math.sign`

与えられた数値の符号に応じた数値を返す関数です。具体的には、正の数なら1が、+0なら+0が、-0なら-0gag、負の数なら-1が返ります。これだけなのでもちろん誤差はありません。

以上で全ての`Math`関数に触れました。

### `Math`の定数

`Math`からはいくつかの定数も提供されています。具体的には`Math.E`, `Math.LN10`, `Math.LN2`, `Math.LOG10E`, `Math.LOG2E`, `Math.PI`, `Math.SQRT1_2`, `Math.SQRT2`です。

例として`Math.SQRT2`の仕様（[20.3.1.8 Math.SQRT2](https://tc39.es/ecma262/#sec-math.sqrt2)）を引用します。

> The Number value for the square root of 2, which is approximately 1.4142135623730951.
>
> （訳）2の平方根を表す数値であり、その値はおよそ1.4142135623730951です。

一見すると“approximately 1.4142135623730951”が定義のように見えますが、そうではありません。実は“The Number value for the square root of 2”の方が定義です。[6.1.6.1 The Number Type](https://tc39.es/ecma262/#number-value) によれば、「The Number value for 実数」という言葉は「その実数に（最近接偶数丸めで）最も近い浮動小数点数」を意味すると定義されています。よって、この定義で`Math.SQRT2`は可能な範囲で最も正確な値を表すことが保証されています。

## おまけ: WebAssemblyの数値計算の正確さ

近年、フロントエンド領域を中心にWebAssemblyが進出してきています。WebAssemblyにも数値計算の命令が備わっているので、この正確性がどうなっているのか調べてみました。[WebAssembly仕様書](https://webassembly.github.io/spec/core/exec/numerics.html#floating-point-operations)には“Floating-point arithmetic follows the IEEE 754-2019 standard”と書かれています。また、丸めモードは常に最近接偶数への丸めです。

fsub・fmulといった具体的な四則演算命令の定義には、結果について“rounded to the nearest representable value.”とあり、JavaScritptの場合と同じ保証がされていることが分かります。

WebAssemblyに組み込まれている一番難しい計算はおそらく[fsqrt](https://webassembly.github.io/spec/core/exec/numerics.html#op-fsqrt)ですが、これについては“return the square root of 𝑧.”とだけ書かれており、制度について書かれていません（implementation-approximatedとは異なり、制度の保証をしないと書かれているわけでもありません）。

IEEE 754仕様は読んでいないので二次情報からの推測ですが（正確な情報をお持ちの方はぜひ教えてください！）平方根はIEEE 754における基本演算に含まれていることから、WebAssemblyにおける平方根は最大限の正確性を持つことが期待されます。正確な平方根が欲しい場合はJavaScriptではなくWebAssemblyで計算すべきかもしれません。

## まとめ

この記事ではJavaScriptにおける数値計算における正確性について言語仕様を基に開設しました。JavaScriptでは、四則演算よりも複雑な演算はほぼ全てimplementation-approximatedとなっており、制度の保証がありません。我々は`Math.sin`が常に0.5を返すかもしれない世界でプログラミングをしているのです。

- **Q.** 仕様は分かったけど実際の処理系ではどうなの？
- **A.** 筆者はあまり興味がないので、読者の演習問題とします。
