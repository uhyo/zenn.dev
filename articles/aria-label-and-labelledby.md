---
title: "aria-labelとaria-labelledbyを併用する場合とは"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["html", "waiaria", "アクセシビリティ"]
published: true
---

アクセシビリティを考慮したマークアップをした経験がある方は、`aria-label`や`aria-labelledby`についてご存じでしょう。これらは、要素にラベル付けするためのWAI-ARIAプロパティです。多くの場合、要素のアクセシブルな名前 (*accessible name*) を決めるために使われます。

`aria-label`はラベルを直接文字列として指定するプロパティで、`aria-labelledby`はIDを通じて他の要素をラベルとして指定するプロパティです。その使い分けについては、仕様書[^wai_aria_13]で以下のように説明されています。つまり、可能な場合は`aria-labelledby`に寄せるべきだということです。

[^wai_aria_13]: [https://www.w3.org/TR/wai-aria-1.3/](Accessible Rich Internet Applications (WAI-ARIA) 1.3
W3C First Public Working Draft 23 January 2024)

> If the label text is available in the DOM (i.e. typically visible text content), authors SHOULD use aria-labelledby and SHOULD NOT use aria-label.
>
> （訳）もしラベルテキストがDOM内に（典型的には可視テキストコンテンツとして）存在する場合、著者は`aria-labelledby`を使うべき (SHOULD) であり、`aria-label`を使うべきではありません (SHOULD NOT)。

## aria-labelとaria-labelledbyの併用

ところで、`aria-label`と`aria-labelledby`を併用することはできるのでしょうか。適当に検索するとそんなことすべきでないというような回答が得られますが、ここで知りたいのはすべきかどうかではなく、できるのかどうかです。

[MDN](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-label)を紐解くと次のような記載があります。

> Don't use both on the same element because `aria-labelledby` will take precedence over `aria-label` if both are applied.
>
> （訳）両方を同じ要素に使わないでください。両方が使用された場合、`aria-labelledby`が`aria-label`を上書きしてしまうからです。

つまり、両方を指定した場合`aria-labelledby`が優先されるため、両方を指定する意味がないということです。

実際に試してみましょう。次のようなHTMLをGoogle Chromeで調べてみます。最近のブラウザの開発者ツールは、アクセシビリティ関連の情報も表示してくれます。この場合、input要素のアクセシブルな名前がどうなっているか調べればいいはずです。

```html
<div id="label1">ラベルです</div>
<p>
  <input type="text" aria-label="aria-labelです" aria-labelledby="label1" />
</p>
```

Google Chromeの開発者ツールで調べると、次の画像のような情報が表示されます。

![開発者ツールのスクリーンショット。以下の内容が表示されている。Nameが"ラベルです"と計算されたこと。aria-labelledbyがdiv#label1を示しており、ラベルの値が"ラベルです"であること。aria-labelがラベルとして"aria-labelです"を指定されていることも表示されているが、打消し線が引かれておりaria-label属性の値が採用されていないことを示している。](/images/aria-label-and-labelledby/chrome-devtools-1.png)

この画像から分かるように、input要素のアクセシブルな名前は確かに`aria-labelledby`に従っています。しかも、aria-labelには打消し線が引いてあり、その値が採用されていないことが示されています。

MDNに書かれているように、`aria-label`と`aria-labelledby`を同じ要素に対してした場合は`aria-labelledby`が採用され、`aria-label`の値が使われないことが確かめられました。

## React Ariaさん！？

しかし、この記事はここでは終わりません。筆者がこの記事を書こうと思ったのは、業務で[React Aria](https://react-spectrum.adobe.com/react-aria/)を使っていた際にあることに気付いたからです。

例えば、React Aria ComponentsのDatePickerコンポーネントを使った場合、レンダリング結果には次のようなHTML断片が含まれます。

```html
<div
  role="spinbutton"
  aria-valuenow="2024"
  aria-valuetext="2024"
  aria-valuemin="1"
  aria-valuemax="9999"
  id="react-aria6233036342-:rs:"
  aria-label="年, "
  aria-labelledby="react-aria6233036342-:rs: react-aria6233036342-:rc:"
  contenteditable="true"
  spellcheck="false"
  autocorrect="off"
  enterkeyhint="next"
  inputmode="numeric"
  tabindex="0"
  class="react-aria-DateSegment"
  data-rac=""
  data-type="year"
  aria-describedby="react-aria-description-0"
>
  2024
</div>
```

真ん中あたりを見ると、`aria-label`と`aria-labelledby`が両方指定されていることが分かります。何でこんなことをしているのか？　というのがこの記事の本題です。

React Ariaのアクセシビリティ関連の実装は非常に優れていることが知られていますから、ミスではないはずです。しかし、先ほど調べたように、この場合は`aria-labelledby`しか使われないはずです。では、`aria-label`は何のために指定されているのでしょうか。

ここでは、`arial-labelledby`にはスペース区切りで2つのIDが指定されています。このように複数の要素を指定した場合、ラベルとしては各要素のテキストが連結されたものになります。

そして、1つ目のIDをよく見てください。すると、1つ目のIDは自分自身を指定していることが分かります。

結論から言ってしまえば、**aria-labelledbyで指定するIDとして自分自身を指定した場合、自分自身が表すラベルとしてaria-labelの値が使われる**のです。`aria-labelledby="自分自身のID 他の要素のID"`とした場合、自分自身の`aria-label`と、他の要素のラベルを結合したものがアクセシブルな名前になります。

つまり、`aria-label`と`aria-labelledby`がこのように併用されているのは、**他の要素由来のラベルとただの文字列のラベルを結合して使うためのハック**であると言えます。

上の例を一部再掲します。

```html
<div
  <!-- ... -->
  id="react-aria6233036342-:rs:"
  aria-label="年, "
  aria-labelledby="react-aria6233036342-:rs: react-aria6233036342-:rc:"
  <!-- ... -->
>
```

`aria-labelledby`で指定された1つ目のIDは自分自身です。2つ目のIDが示す要素は実は以下のものであり、ラベルとしては`Date`であることが分かります。よって、このdiv要素のアクセシブルな名前は`年, Date`となります。

```html
<span class="react-aria-Label" id="react-aria6233036342-:rc:">Date</span>
```

## 仕様書で確かめる

以上では、React Ariaで使われているテクニック（というかハック）を紐解くことで、`aria-labelledby`で自分自身のIDを指定した場合は、本来無視されるはずの`aria-label`の値を拾うことができることが分かりました。たしかに、単純に考えると無限ループになってしまうので何らかの特殊な挙動は必要です。

ということで、この挙動がちゃんと仕様に沿ったものであることを確かめましょう。今回参照するのは、[Accessible Name and Description Computation 1.2
W3C Working Draft 02 August 2024](https://www.w3.org/TR/accname-1.2/)です。

この仕様には、`aria-label`等からアクセシブルな名前の値を計算するアルゴリズムが示されています。具体的には、[4.3.2 Computation steps](https://www.w3.org/TR/accname-1.2/#computation-steps)です。

特に、ノードの`aria-labelledby`プロパティに基づいて名前を計算するところを引用します。

> LabelledBy: Otherwise, if the current node has an aria-labelledby attribute that contains at least one valid IDREF, and the current node is not already part of an ongoing aria-labelledby or aria-describedby traversal, process its IDREFs in the order they occur:
> 
> 1. Set the accumulated text to the empty string.
> 2. For each IDREF:
>     1. Set the current node to the node referenced by the IDREF.
>     2. LabelledBy Recursion: Compute the text alternative of the current node beginning with the overall Computation step. Set the result to that text alternative.
>     3. Append a space character and the result to the accumulated text.
> 3. Return the accumulated text if it is not the empty string ("").

> （訳）LabelledBy: さもなければ、もしcurrent nodeが少なくとも1つの有効なIDREFを含むaria-labelledby属性を持ち、かつcurrent nodeがすでに進行中のaria-labelledbyまたはaria-describedbyのトラバーサルの一部でない場合、そのIDREFを指定された順序で処理する。
>
> 1. accumulated textを空の文字列に設定する。
> 2. 各IDREFについて:
>    1. IDREFで参照されるノードをcurrent nodeに設定する。
>    2. LabelledBy Recursion: current nodeの代替テキストを計算する（Computationステップから行う）。resultを計算された代替テキストとする。
>    3. accumulated textにスペース文字とresultを追加する。
>  3. accumulated textが空文字列でなければ、それを返す。

特に2-2の部分で、`aria-labelledby`で指定された要素に対してラベルを計算する再帰的な処理があります（Computationステップというものが言及されていますが、これは計算処理全体を指すものです）。

ポイントは、「current nodeがすでに進行中のaria-labelledbyまたはaria-describedbyのトラバーサルの一部でない場合」という条件です。この条件により、2-2で自分自身へ再帰したときの無限ループを回避しています。この場合、*LabelledBy:*ステップの処理に入りませんから、自分自身に再帰したときは`aria-labelledby`が使われないことになります。

詳細は省きますが、アルゴリズムには上の*LabelledBy*という分岐の次に、input要素（など）の入力値を使用する*Embedded Control*ステップ、そしてその次に`aria-label`を参照する*AriaLabel*分岐が続きます。よって、再帰してきて*LabelledBy*分岐に入れなかった場合、そのあとの*AriaLabel*分岐に入って`aria-label`の値が使われることになります。

以上のことから、`aria-labelledby`で自分自身のIDを指定した場合には自身の`aria-label`の値が使われるという挙動が仕様に沿ったものであることが分かりました。

余談ですが、このアルゴリズムで`aria-labelledby`の処理が`aria-label`よりも先に来ていることにより、`aria-label`よりも`aria-labelledby`が優先されるということが示されています。よく見ると3で「accumulated textが空文字列でなければ、それを返す」とありますから、`aria-labelledby`のルールに従って計算しても空文字列しか得られなかった場合は、`aria-label`のほうが使われることになります。

フォールバックとしてはありかもしれませんが、普通は`aria-labelledby`でわざわざIDを指定する以上はそのIDが指す要素に適切なラベルがあるでしょうから、この挙動を活用することはあまり無さそうです。

## まとめ

この記事では、`aria-label`と`aria-labelledby`を併用するという実装をReact Ariaが行っていたことをきっかけに、その場合の挙動について調べて解説しました。

結論としては、`aria-labelledby`で自分自身のIDを指定した場合には自分自身の`aria-label`の値が使われるため、このケースにおいては併用する意味があることが分かりました。これは、仕様に沿った挙動であり、他の要素由来のラベルとただの文字列のラベルを結合して使うためのハックとして使われています。