---
title: "<label>で<button>を囲んでいるときにclickイベントが2回発火する問題の原因と対策"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "dom"]
published: true
---

皆さんこんにちは。今回は、最近筆者が遭遇した、`<label>`で`<button>`を囲んでいるときにclickイベントが2回発火することがある問題について解説します。

さっそくですが、こちらのCodePenをご覧ください。

@[codepen](https://codepen.io/uhyo-the-flexboxer/pen/LEPeVQY?default-tab=output,html,js)

ここでは、0と書かれたボタンが表示されています。このボタンは1回クリックすると数字が1増えるようになっています。

しかし、ボタンに表示されている数字をクリックすると、数字が2増えてしまいます。これは、clickイベントが2回発火しているためです。それ以外の部分（ボタンの端や、ラベル）をクリックした場合は数字が1増えます。

実装のHTMLとJavaScriptは以下のとおりです。

```html
<p>数字をクリックするとonClickが2回発火する！！！</p>
<div>
<label>
  ラベルのテスト
  <button type="button"></button>
</label>
</div>
```

```javascript
let count = 0;

const writeButtonContent = () => {
  const button = document.querySelector('button');
  const span = document.createElement('span');
  span.textContent = String(count);
  button.replaceChildren(span);
}

writeButtonContent();

document.querySelector('button').addEventListener('click', () => {
  console.log('click');
  count++;
  writeButtonContent();
})
```

これを見て、clickイベントが2回発火する現象がなぜ起きるのか説明できる人は、なかなかHTMLやDOMに詳しい人です。筆者も、理解するのに少し時間がかかりました。

ということで、この記事ではこの現象の原因を説明し、対策を紹介します。

## 2回clickイベントが発火する原因

JavaScriptが動作した後のDOM構造は次のようになっています（p要素とdiv要素は今回関係ないので省略）。

```html
<label>
  ラベルのテスト
  <button type="button"><span>0</span></button>
</label>
```

![DOM構造を表す図。label要素の下にbutton要素、その下にspan要素、その下にテキストノードの0がある。](/images/label-button-onclick-twice/dom-tree-1.png)
*初期状態のDOM構造*

2回clickイベントが発火するのは、button要素の中のspan要素の部分がクリックされたときです。デバッグしてみると、1回目のclickイベントはspan要素に対して発火しており、2回目のclickイベントはbutton要素に対して発火していることがわかります。

ポイントは、**イベントバブリング**と**label要素の挙動**にあります。また、今回のサンプルでclickイベント時に**DOM書き換え**が行われていることも関係があります。

まず、1回目のclickイベントはspan要素で発生します（言い換えれば、イベントの`target`はspan要素です）。そして、イベントバブリングによりその親のbutton要素でclickイベントのイベントハンドラが処理されます。

![span要素がイベントの発火元であり、イベントバブリングによりbutton要素に到達したことを表す図。](/images/label-button-onclick-twice/dom-tree-1-event.png)
*イベントバブリングの様子*

この時、DOM書き換えにより、DOMは次のように書き換えられます。

```html
<label>
  ラベルのテスト
  <button type="button"><span>1</span></button>
</label>
```

特に、button要素の中身が`<span>1</span>`になりましたが、これはspan要素ごと新しく作られています。つまり、元の`<span>0</span>`のspan要素はDOMツリーから外されています。

そしてイベントの処理は続きます。

![イベントの発火元であるspan要素がDOMツリーから切り離され、新たにbutton要素の下にspan要素と1のテキストノードが作られたことを表す図。さらに、イベントバブリングがlabel要素以降へ進んでいる。](/images/label-button-onclick-twice/dom-tree-2-event.png)
*書き換え後のDOMとイベントバブリングの様子*

label要素の挙動は、（for属性が無い場合は）label要素がクリックされたとき、その中に含まれるフォームコントロール[^labelable_element]に対してclickイベントを発火するというものです。これが、button要素で2回目に発火するclickイベントです。

[^labelable_element]: 正確には[labelable element](https://html.spec.whatwg.org/multipage/forms.html#category-label)

しかし、普通にbutton要素をlabel要素で囲っただけでは、このような問題は発生しません。この現象は、DOM書き換えを行ったことも原因のひとつです。

つまり、label要素が1回目のclickイベントを処理するするとき、「これがもともとbutton要素がクリックされたことで発生したclickイベントならば、追加でclickイベントを発火させる必要はない」という判断をしているはずです。これにより、通常の状況で単にbutton要素をクリックしても、2回clickイベントが発火することはありません。

今回は、1回目のclickを発火させたspan要素がこの時点でDOMツリーから外れているため、発火元はbutton要素の中ではないという判断がされていると考えられます。このため、label要素が2回目のclickイベントを発火させてしまうのです。

以上が、1回ボタン（の中身）をクリックしただけで、button要素のclickイベントが2回処理されてしまう理由です。clickイベントでこのようにDOMを書き換えることは、いわゆるトグルボタン的にbutton要素を使っていた場合に起こりがちですね。筆者はReactアプリケーションでこの問題に遭遇しました。

## 仕様を確かめる

この問題は、label要素のクリック処理中に発火元がbutton要素ではないと勘違いされてしまったために、clickイベントが2回処理されてしまうという問題でした。

では、この挙動に根拠はあるのでしょうか。HTML文書におけるブラウザの挙動は、[HTML仕様書](https://html.spec.whatwg.org/multipage/forms.html#the-label-element)で定義されています。もちろん、label要素の定義もあります。仕様書の記述を確認してみましょう。

:::message
この記事では、HTML仕様書のコミット`8e9551669105bcf08dffcb6f9e1cfc0ef5f5d366`版を参照しています。この記事で引用した部分がこれ以降の変更により古くなっている可能性がありますので、ご注意ください。
:::

いきなり核心に迫る部分を引用します。

> The label element's exact default presentation and behavior, in particular what its activation behavior might be, if anything, should match the platform's label behavior. The activation behavior of a label element for events targeted at interactive content descendants of a label element, and any descendants of those interactive content descendants, must be to do nothing.

（訳）label要素の厳密なデフォルト表示や挙動は、特にlabel要素のactivation behaviorが何であるかは、プラットフォームにおけるラベルの挙動と一致すべきです。イベントのターゲットがlabel要素のinteractive contentである子孫、もしくはそれらのさらに子孫である場合には、そのイベントに対するlabel要素のactivation behaviorは何もしないことでなければなりません。

ここで出てきた[activation behavior](https://dom.spec.whatwg.org/#eventtarget-activation-behavior)は、要するに仕様で定められた要素のデフォルトの挙動であると考えてください。例えば、リンクをクリックしたらページ遷移するとか、`<button type="submit">`であればクリックされたときにフォームが送信されるとかがactivation behaviorの一例です。

これを読むと分かるように、label要素のactivation behaviorは具体的なアルゴリズムとして定義されていません。そうではなく、「プラットフォームにおけるラベルの挙動と一致すべき」とされています。これについては、例として次のように言及されています。

> For example, on platforms where clicking a label activates the form control, clicking the label in the following snippet could trigger the user agent to fire a click event at the input element, as if the element itself had been triggered by the user:
>
> ```html
> <label><input type=checkbox name=lost> Lost</label>
> ```

（訳）例えば、ラベルをクリックするとフォームコントロールがアクティブになるプラットフォームでは、次のスニペットのラベルをクリックすると、ユーザーエージェントによりinput要素にclickイベントが発火されるということが考えられます。このイベントは、ユーザーがその要素自体をトリガーしたかのように扱います。

この例で述べられている挙動が、PC等における一般的なラベルの挙動です。筆者が確認した限りでは、PCでもスマートフォンでもこの挙動です。ただし、これが絶対というわけではなく、仕様書では他にありえる挙動として「フォーカスが当たるだけ」や「何も起きない」ということも考えられると述べられています。

今回問題となっているのはlabel要素のactivation behaviorで唯一mustとされていたところです。訳だけ再掲します。

> イベントのターゲットがlabel要素のinteractive contentである子孫、もしくはそれらのさらに子孫である場合には、そのイベントに対するlabel要素のactivation behaviorは何もしないことでなければなりません。

button要素はinteractive contentに該当するので、「button要素の中のspanをクリックした場合」はこのケースに該当します。このため、通常であればlabel要素のactivation behaviorは何もしないはずです。

仕様では、この判定（イベントのターゲットがinteractive contentかどうか）をいつ行うのかについて、仕様では明記されていません。

ブラウザの実装を確認したわけではないので推測ですが、activation behaviorが起動したタイミングで判定も行われていると考えられます。Activation behaviorの起動タイミングは、[仕様で定義されているように](https://dom.spec.whatwg.org/#dispatching-events)、イベントバブリングが全部完了した最後です[^preventDefault]。これであれば、上述の通り判定時にはすでにターゲットのspan要素はDOMツリーから外れているため、activation behaviorはbutton要素のclickイベントを発火してしまうという挙動になりますね。

[^preventDefault]: イベントバブリングのどのタイミングで`preventDefault`を呼び出してもactivation behaviorをキャンセルできることを思い出しましょう。そうなると、イベントバブリング完了後でないといけませんね。

ただ、判定をいつ行うのかは現状仕様で厳密に定義されていません。そのため、この記事の状況でも判定を正しく行い、2回目のclick発火を行わないようなブラウザがあったとしても仕様違反にはならないでしょう[^browser_diverse]。

[^browser_diverse]: もっとも、もし実際にブラウザ間で挙動が異なる状態になったら、ブラウザの挙動が統一されて仕様が厳密化される方向に進むと思いますが。

## 2回clickイベントが発火する現象の対策

問題の原因が一応分かったところで、対策を紹介します。可能な対策は主に2つあります。

### preventDefaultを使う

Activation behaviorはpreventDefaultによってキャンセルすることができます。そのため、button要素のclickイベントハンドラでpreventDefaultを呼び出すことで、buttonの親にlabel要素があってもそのactivation behaviorをキャンセルできます。

```diff javascript
 document.querySelector('button').addEventListener('click', (event) => {
+  event.preventDefault();
   console.log('click');
   count++;
   writeButtonContent();
})
```

ちなみに、preventDefaultの代わりにstopPropagationを使っても止められません。なぜなら、stopPropagationはイベントバブリングの最中のイベントリスナーの処理を止めるものであって、イベントバブリングの後に発火するactivation behaviorを止めるものではないからです。

基本的に、この方法がデメリットが少なく、多くの場合に適切な対策です。実際にはbutton要素がコンポーネントであるようなケース（親にlabel要素があるかどうか不明）を考えると、親にlabel以外が来た場合に意図せずそのactivation behaviorをキャンセルしてしまう可能性が考えられますが、HTML仕様ではinteractive contentsをネストさせてはいけないことになっているので、実用上それで困る場合はあまり無さそうです。

### `pointer-events: none;`を使う

次のように、buttonの中身に対して`pointer-events: none;`を指定するのも対策になります。

```css
button span {
  pointer-events: none;
}
```

このようにした場合、ボタンのどこをクリックしても、clickイベントのtargetがbutton要素になります。

今回の事例では、発火元がDOMツリーから取り除かれることが問題でしたが、button要素自身はDOMツリーに残っていますから、label要素のactivation behavior時の判定が正しく行われます。このため、2回目のclickイベントが発火されなくなります。

これもなかなか良い方法ですが、button要素が実際にはコンポーネントであり、その中身が不明な場合は注意が必要です。子孫要素で`pointer-events: auto;`に戻されてしまうかもしれないからです。

## まとめ

この記事では、`<label>`で`<button>`を囲んでいるときにclickイベントが2回発火する問題について解説しました。

この問題は、button要素のイベントハンドラの最中にbutton要素の中身を書き換え、発火元（target）の要素がDOMツリーから外れてしまった場合に起こるものであることが分かりました。実際のReactアプリケーションなどでも、このようなシチュエーションは起こりがちです。

この問題に対する対策として、preventDefaultを使う方法と`pointer-events: none;`を使う方法を紹介しました。場合によって使い分けると良いでしょう。

もし皆さんの手元に中身が書き換わるボタンコンポーネントがあったら、`<label>`で囲んでみてください。もしかしたらclickが2回発火しているかもしれませんよ。