---
title: "React 18 alpha版発表まとめ"
emoji: "🌘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

先日、[The Plan for React 18](https://reactjs.org/blog/2021/06/08/the-plan-for-react-18.html)という記事が React チームから発表されました。これは React の次期メジャーバージョンである React 18 で予定されている変更や新機能を紹介するとともに、React 18 の alpha 版の公開を知らせるものです。この記事自体に技術的なトピックは載っておらず、それらは[React 18 Working Group](https://github.com/reactwg/react-18)という新設されたリポジトリに Discussion としてまとめられています。

本記事では、今回あった発表のポイントを厳選してお伝えします。ポイントを絞ってお伝えするため載せる情報は取捨選択しています。隅々まで理解したいという方は原文か他の記事を参照しましょう。

## アップグレードの簡単さ

React 17 の際もそうでしたが、最近の React は「**簡単にアップデートできる**」ことをたいへん重要視しており、React 18 においてもユーザーが簡単にアップグレードできるように準備されています。以下に引用するように、ほとんどのユーザーにとっては React 18 に移行するだけなら 1 日もかからないだろうと見積もられています（React 18 の新機能を活用するために追加の努力が必要になる可能性はあります）。

> Since concurrency in React 18 is opt-in, there are no significant out-of-the-box breaking changes to component behavior. **You can upgrade to React 18 with minimal or no changes to your application code, with a level of effort comparable to a typical major React release.** Based on our experience converting several apps to React 18, we expect that many users will be able to upgrade within a single afternoon.
>
> [The Plan for React 18](https://reactjs.org/blog/2021/06/08/the-plan-for-react-18.html)から引用

## Concurrent Rendering

2 年ほど前から React は「Concurrent Mode」の開発を公言していましたが、今回の発表では「**Concurrent Rendering**」としてリブランディングされました。これは上述の「アップグレードの簡単さ」を助けるためです。これまで準備されていた「mode」という概念は 0 か 100 かであるためスムーズな移行の妨げになると考えられました。そこで、React 18 に向けて API や内部のアーキテクチャがリデザインされ、concurrent rendering が必要な場面でだけ自動的にオプトインされる設計になりました。これにより、従来 Concurrent Mode として知られていた機能の中でも必要なものから徐々に入れていくという段階的なオプトインが可能になっています。

> If you’ve been following our research into the future of React (we don’t expect you to!), you might have heard of something called “concurrent mode” or that it might break your app. In response to this feedback from the community, we’ve redesigned the upgrade strategy for gradual adoption. Instead of an all-or-nothing “mode”, concurrent rendering will only be enabled for updates triggered by one of the new features. **In practice, this means you will be able to adopt React 18 without rewrites and try the new features at your own pace.**
>
> [The Plan for React 18](https://reactjs.org/blog/2021/06/08/the-plan-for-react-18.html)から引用

## Automatic Batching

[Automatic batching for fewer renders in React 18](https://github.com/reactwg/react-18/discussions/21)で紹介されています。

従来の React では、（イベントハンドラの中で同期的に実行される場合を除いて）連続して複数回ステートを更新すると複数回再レンダリングが行われていましたが、このような場合に再レンダリングが 1 回しか起こらないようになります。助かりますね。

これはリスクが少ないと判断されたのか、オプトインではなくデフォルトで有効になります。オプトアウト用の API として`flushSync`関数が`react-dom`に追加されます。

## SSR と Suspense

React の Suspense は従来 SSR で使えませんでしたが、React 18 では Suspense の機能強化と合わせて SSR の Suspense サポートが改善されます。ただし、従来 SSR で使われてきた`renderToString`では Suspense サポートが限定的で、最低限「エラーが起こらない」程度です。これは同期的な API である以上、Suspense の利点が出ないのは当然のことですね。さらなる Suspense サポートを SSR で得るためには新しいストリーミング API である`pipeToNodeWritable`をオプトインします。

SSR における Suspense サポートは[New Suspense SSR Architecture in React 18](https://github.com/reactwg/react-18/discussions/37)で説明されています。ポイントは、React 18 では SSR 時においてパイプライン的な挙動がサポートされるということです。

### Suspense と HTML ストリーミング

大抵の場合、SSR されるデータにはサーバー内で例えばデータベースなどから取得したデータが含まれます。従来の SSR というのはワンパスであり、必要なデータが全て出揃わないと HTML の生成を開始できませんでした。本来データを必要としない部分の SSR まで、データ取得に引きずられて遅延してしまうことがひとつの問題意識として挙げられています。

> One problem with SSR today is that it does not allow components to “wait for data”. With the current API, by the time you render to HTML, you must already have all the data ready for your components on the server. This means that **you have to collect all the data on the server before you can start sending any HTML to the client.** This is quite inefficient.
>
> [New Suspense SSR Architecture in React 18](https://github.com/reactwg/react-18/discussions/37) から引用

React 18 では、`<Suspense>`に囲まれた内部のコンポーネントたち（データ取得に時間がかかるコンポーネントたち）を SSR する場合、まずフォールバックコンテンツ（ローディング中に表示するコンテンツ）の HTML を出力して次に進みます。そのため、最初はローディング中の画面を SSR した HTML がクライアントに送られることになります。これによって、クライアントはサーバー内でのデータ取得の完了を待たずにページを見ることができ、「データ取得に引きずられて関係ない部分の SSR が遅延してしまう」問題が解消されます。

さらに、SSR はそれで終了ではなく、サーバー側でデータが読み込まれた時点で`<Suspense>`内のレンダリング結果が SSR され、**レンダリング済みのフォールバック HTML を読み込み完了後の内容で置き換えるための script タグが配信されます**。つまり、「データの読み込みが終わったら DOM を再レンダリングする」という処理すら、クライアント側の React ランタイムに任せるのではなくサーバー側で担当することになります。そのために、それ専用に最適化された生 DOM を操作するスクリプトが送られてきます。Discussion では PoC 的に次のような例が示されています。データの読み込みが終わった時点で次のような HTML と JavaScript がサーバーから送られてきて、前に送られてきた HTML （下の例では `#sections-spinner`)を更新します。

> The story doesn’t end here. When the data for the comments is ready on the server, React **will send additional HTML into the same stream, as well as a minimal inline &lt;script&gt; tag to put that HTML in the “right place”**:
>
> ```html
> <div hidden id="comments">
>   <!-- Comments -->
>   <p>First comment</p>
>   <p>Second comment</p>
> </div>
> <script>
>   // This implementation is slightly simplified
>   document
>     .getElementById("sections-spinner")
>     .replaceChildren(document.getElementById("comments"));
> </script>
> ```
>
> [New Suspense SSR Architecture in React 18](https://github.com/reactwg/react-18/discussions/37) から引用

つまり、SSR の一環としてワンタイム・インラインな JavaScript の生成まで行うことで、HTML のストリーミングのみを通じて段階的なローディングの機構を実現するのが React 18 の SSR です。SSR なのに JavaScript を使うというのは受け入れ難い方もいるかもしれませんが、パフォーマンスのためのアプローチとして確かに理にかなっています[^ad_castella]。

[^ad_castella]: 宣伝ですが、筆者のライブラリ[Castella](https://github.com/uhyo/castella)でも Shadow DOM 付きの Custom Elements で SSR をサポートするためにこのアプローチを採用していました。このユースケースに関しては Declarative Shadow DOM があれば不要なのですが。

ただ、ニュースサイトなどのように、パフォーマンスだけでなく SEO のために SSR を行なっているケースでは、SSR で JavaScript を含む HTML が出力されると SEO への影響が心配されます。この懸念については[Discussion 内で回答があります](https://github.com/reactwg/react-18/discussions/37#discussioncomment-842581)。結論としては、データが出揃ってから HTML を出力する（従来と同じ方式に）することも簡単に（アプリの内部実装を変えずに`renderToNodeWritable`の使い方を変えるだけで）可能であるため、目的によって適切なものを選択することができます。さらに言えば、クライアントがクローラであると思われる場合にのみストリーミングの方式をスイッチすることすら可能でしょう。

ちなみに、ひとつ理解していただきたいのは、このように SSR に JavaScript を混ぜこむことは SSR における本質ではなく、「HTML が前から順に構文木を記述する方式しかサポートしていない」という問題へのワークアラウンドでしかないということです。これに思うところがあるのか、Dicsussion 内でもわざわざこのことが太字で書かれています。

> **Unlike traditional HTML streaming, it doesn’t have to happen in the top-down order.**
>
> [New Suspense SSR Architecture in React 18](https://github.com/reactwg/react-18/discussions/37) から引用

つまり、HTML 自体に「すでに出力された木を書き換える」ことができる静的な文法があれば、JavaScript に頼る必要が無くなります。今はそのような機能が HTML にないため、仕方なく JavaScript を使っているのです。個人的に、HTML 自体にこの機能を加えるようなプロポーザルを Facebook が出してくれないかなと思っています。文書を上から下に順番にしか表現できない現在の HTML がもはや現代の需要に対して力不足であり、進化が必要なのではないでしょうか。

### 選択的ハイドレーション

ハイドレーションの仕組みも React 18 では改善されています。ハイドレーションは、イベントハンドラなどを SSR された HTML にアタッチすることです。従来の方式では、ハイドレーションを行うためにまずページ全体をクライアント側で再度レンダリングする必要があります。つまり、ページをレンダリングするために必要な JavaScript コードが全部読み込まれないとハイドレーションを行えないという問題がありました。React 18 ではこの点を改善する仕組みとして**選択的ハイドレーション** (Selective Hydration) が導入されます。

具体的には、`React.lazy`で code splitting・遅延ロードされたコンポーネントがある場合、それ以外の（code splitting されていない）部分の JavaScript が読み込み完了した時点でそれらだけが先にハイドレーションされます。もちろん、遅延ロードされたコンポーネント用のコードがその後読み込まれれば、その部分が追加でハイドレーションされます。

なお、選択的ハイドレーションは`<Suspense>`を境界として行われます。つまり、個々のコンポーネントレベルで選択的ハイドレーションが行われるのではなく、`<Suspense>`で区分けされた部分が一括で行われます。これについては次のように説明されています。つまり、`<Suspense>`にすでに「中身がまだ読み込まれていないかもしれない領域の境界」という意味付けがされており、これは「中身がまだハイドレーションされていないかもしれない領域の境界」としても有効に働くことが期待されています。おそらく、HTML が SSR されているがまだハイドレーションが済んでいない部分は、クライアントの JavaScript コードから見ればまだ読み込みが済んでいない部分として見えるのでしょう。

> Note: You might be wondering how your app can work in this not-fully-hydrated state. There are a few subtle details in the design that make it work. For example, instead of hydrating each individual component separately, hydration happens for entire <Suspense> boundaries. Since <Suspense> is already used for content that doesn't appear right away, your code is resilient to its children not being immediately available.
>
> [New Suspense SSR Architecture in React 18](https://github.com/reactwg/react-18/discussions/37) から引用

おまけに、React 18 では「まだハイドレーションされていない部分で click などのイベントが発生した場合、ハイドレーション後にイベントをリプレイする」という機能が導入されるようです。これにより、ハイドレーション前にユーザーがページを触った場合の挙動も通常は心配なくなります。とても嬉しいですね。その上、イベントが発生した場合はその部分のハイドレーションを優先するという優先度制御の機構までついています。

### React Server Components との関係

今回 React の SSR が改善されるということで、以前に発表された React Server Components との関係を疑問に思う React ユーザーがたいへん多いようです。結論を一言で言うと、Server Components は初期レンダリング（SSR）だけでなく再レンダリングの際にも働くという点で、SSR とは根本的に異なる仕組みです。両者は相補的 (complementary) なもの、つまり別々の恩恵をもち併用できるものであると説明されています。

## Concurrent Rendering 関係

先ほども触れた通り、従来 Concurrent Mode と呼ばれてきた機能や仕様変更も React 18 の解説に改めて含まれています。

例えば[Behavioral changes to Suspense in React 18](https://github.com/reactwg/react-18/discussions/7)では、React 18 では従来あった「一度レンダリングが開始された（関数コンポーネントの場合関数が呼ばれた）コンポーネントは DOM のマウントまで行われる」という保証が撤廃されることが説明されています[^note_rendering_finish_guarantee]。これは Concurrent Mode の発表時から言われていたことなので特に新しいわけではありません。

[^note_rendering_finish_guarantee]: 関数コンポーネントが呼ばれたがそのレンダリングは中断されて DOM にマウントされないということが起こり得るようになります。

前述の通り基本的には Concurrent Rendering を用いる機能をオプトインした際にこのことが顕在化しますが、`Suspense`を用いていた従来のアプリケーションも一部影響を受けると言うことがこの Discussion で説明されています。これも既存のアプリケーションへの影響が小さいと思われることからオプトイン方式にはなっていません。

### `startTransition`

Concurrent Rendering 関係で従来の Concurrent Mode の説明からアップデートがあったのが`startTransition`と言う新しい API です。以前は`useTransition`というフックを通してこの機能が提供されていましたが、フックを介さずに直接`startTransition`が React からエクスポートされるようになりました（従来の`useTransition`も引き続きサポートされます）。この点に関しては、フックに縛られずに`startTransition`が使えるようになったことでトランジションを扱うライブラリを作りやすくなるため大歓迎です。

`startTransition`にまつわる問題意識は次に引用する段落によくまとまっています。

> **Until React 18, all updates were rendered urgently.** This means that the two state states above would still be rendered at the same time, and would still block the user from seeing feedback from their interaction until everything rendered. What we’re missing is a way to tell React which updates are urgent, and which are not.
>
> [New feature: startTransition](https://github.com/reactwg/react-18/discussions/41) から引用（強調は筆者）

つまり、従来 React は全てのステート更新を ASAP で画面に反映しようとしていましたが、実はステート更新にはすぐ画面に反映すべきものとそうでもないものがあります。これらを区別する手段が`startTransition`によって提供されます。これにより、React のスケジューラはすぐ画面に反映すべきもの（具体例としては controlled component たち）のレンダリングを優先して行うことができ、結果的に UX が向上します。

具体的には、`startTransition`を用いるとステート更新を「トランジション」として扱うことができます（一つまたは複数のステート更新をまとめて一つのトランジションとできます）。トランジションはすぐ画面に反映しなくてもよいステート更新であると扱われます。トランジション内のステート更新により再レンダリングが発生した場合、より優先度が高いタスクが発生した場合はその再レンダリングが中断し破棄されます（前述の保証が React 18 で撤廃されるのはこの動作を可能にするためです）。

> **Updates wrapped in startTransition are handled as non-urgent** and will be interrupted if more urgent updates like clicks or key presses come in. If a transition gets interrupted by the user (for example, by typing multiple characters in a row), React will throw out the stale rendering work that wasn’t finished and render only the latest update.
>
> [New feature: startTransition](https://github.com/reactwg/react-18/discussions/41) から引用（強調は筆者）

従来の Concurrent Mode の説明では「トランジション」とは何かがいまいち明確でなかったのですが、今回の説明ではトランジションが「優先度が低いステート更新」であると明言されたことは注目に値します。おそらく内部的に API デザインやアーキテクチャを洗練させた成果なのでしょう。

> **In a typical React app, most updates are conceptually transition updates.** But for backwards compatibility reasons, transitions are opt-in.
>
> [New feature: startTransition](https://github.com/reactwg/react-18/discussions/41) から引用（強調は筆者）

上に引用した通り、非常に高い優先度 (Discussion 内では“urgent”と説明されています）を持つステート更新は限られており、ほとんどのステート更新はトランジションとして扱って（`startTransition`で囲って）も問題ないはずです。しかしトランジションはオプトアウトではなくオプトインなので、こだわるならばありとあらゆるステート更新をトランジションで囲むことになります。`useMemo`みたいに「とにかく`startTransition`で囲むべき派」と「必要なければわざわざ`startTransition`を書かなくていい派」が現れることが予想されます。楽しみですね。別の方向性としては、トランジションがデフォルトなステート管理ライブラリが台頭しそうですね。今後のステート管理ライブラリの動向で「デフォルトでトランジション」みたいな説明が現れたらこのことを指しています。注視しましょう。

## ReactDOM.render に対する変更

従来 React アプリのエントリーポイントとして`ReactDOM.render` API が使用されてきましたが、React 18 ではこれは Legacy root API として位置付けられます。新しい API は`ReactDOM.createRoot`です。React 18 の新機能を使うためにはまず`ReactDOM.createRoot`に移行する必要があります。

`render`から`createRoot`への以降は、React を 18 にアップグレードする際に行なってしまうべきです。なぜなら、今後 React 18 に対応したサードパーティのライブラリが出てくることが期待されますが、それらが`ReactDOM.render`環境下で動くと期待すべきではないからです。React 18 で`ReactDOM.render`を使うと warning が出ます。React の warning はそれを放置せずすみやかに修正すべきであるということを意味しています。

> So in that sense it is assumed that 18 is createRoot, because render warns and it means you haven't fully completed the migration.
>
> [Replacing render with createRoot](https://github.com/reactwg/react-18/discussions/5)から引用

## StrictMode での useEffect の挙動の変化

React は従来から`StrictMode`というコンポーネントを提供しており、これは“正しく”実装されていないコンポーネントを炙り出すのに有効とされています。例えば、関数コンポーネントの関数をわざと 2 回呼ぶことで、副作用を持ってしまっているコンポーネントを検出できる可能性があります。StrictMode 下で問題が発生したコンポーネントは React のルールに従って実装されたコンポーネントではなく、修正すべきです。

正しく実装されていないコンポーネントは、今問題なくても将来的に問題が発生するかもしれません。関数を 2 回呼ぶことは、直接的には前述の Concurrent Rendering への布石となっていました。Concurrent Rendering では実際に 1 回のレンダリングに対して関数が複数回呼ばれる事象が発生するようになり、React 16.3 というはるか昔からその準備がされていたのです。

React 18 では`StrictMode`に新たな挙動が追加されます。言い方を変えれば、React 18 では“正しい”コンポーネントのルールが追加されるということです。

具体的には、`StrictMode`下では「コンポーネントのマウント時に余計に`useEffect`が発火する」という挙動が加えられます。特に、`useEffect(() => { /* ... */ }, [])`という形のエフェクトであっても複数回走る可能性が生じます。言うまでもなく、これも将来への布石です。具体的には、**Offscreen API**の提供が将来的に予定されており、Offscreen API で使えるコンポーネントであるためには`StrictMode`の新しい挙動にも耐えなければならないと説明されています。

> The main motivation for the new Offscreen API (and the effects changes described in this post) is to allow React to preserve state like this by hiding components instead of unmounting them. To do this React will call the same lifecycle hooks as it does when unmounting– but it will also preserve the state of both React components and DOM elements.
>
> [Adding Strict Effects to StrictMode](https://github.com/reactwg/react-18/discussions/5)から引用

簡単に言えば、これは React コンポーネントに従来あった mount/unmount というライフサイクルに加えて「hidden」という新たな状態を与えるもので、具体的には「レンダリングされた DOM は残っているが画面に表示されていない状態」のコンポーネントを作ることができるようになります[^note_dom]。このとき`useEffect`のサイクルは「mount で発火 →unmount でクリーンアップ」という従来のものから「**表示された**ら発火 →**非表示になった**らクリーンアップ」と再定義されます。コンポーネントは unmount されずに非表示 → 表示と行き来する可能性があるため、たとえ`useEffect(() => { /* ... */ }, [])`だったとしてもひとつのコンポーネントで複数回エフェクトが発火する可能性が生じるのです。この挙動に耐えられるかあらかじめ確かめるために`StrictMode`の新しい挙動が活用できます。このように`useEffect`を再定義する理由は次のように説明されています。React の Effect 観が現れていますね。

> It wouldn’t make sense for an unmounted component to trigger some imperative code (e.g. to position a tooltip). The same is true for a component that’s been hidden. So React needs to tell the component that it is being hidden. How? By calling the same functions that it calls when the component is unmounted (either an effect cleanup functions or `componentWillUnmount`).
>
> [Adding Strict Effects to StrictMode](https://github.com/reactwg/react-18/discussions/5)から引用

[^note_dom]: DOM が残るというのが実際に文書ツリー内に残るのかメモリ上にだけあるのかは情報が無くよく分かりませんでした。まだ未定かもしれません。

Twitter などでは「やばい破壊的変更だ」といった声も見られましたが、実はそうでもありません。まず、すぐに挙動が変わるのは`StrictMode`を使用していて開発環境の場合のみで、あなたのプロダクション環境に即座に影響があるわけではありません（開発環境が壊れてしまう懸念があるかもしれませんが、React の新機能を使うつもりならいずれ直さなければいけないので直しましょう。`StrictMode`を使っていなくても挙動が変わるのは、オプトインの Offscreen API を使うときのみです[^note_react_refresh]。

[^note_react_refresh]: 他の例として、React Refresh への対応も挙げられています。マウント済みのコンポーネントの定義が React Refresh によって書き換えられた場合、`useEffect`のエフェクトが再実行されることがあります。

ただし、**StrictMode を使っていなければあなたは無関係というわけではありません**。React 18 にアップグレードするということは新しい“正しさ”の定義を受け入れるということであり、もし React 18（やそれ以降）の新機能を使っていく気があるならば、**正しくないコンポーネントは最終的には直さなければいけません**。むしろこの機会に`StrictMode`を導入するくらいの勢いがあったほうが良いでしょう。

幸いにも、React は後方互換性や簡単なマイグレーションを重要視していますから、今後 React の新機能を使わないのであれば今動いているコンポーネントが勝手に壊れることはないでしょう。尤も、その場合はわざわざ React 18 にアップグレードしなくても良いのですが。

## Built-in Suspense Cache

現在公開されている alpha 版には含まれていませんが、React 組み込みのキャッシュ機構が導入されて`unstable_getCacheForType`、`unstable_useCacheRefresh`、`<Cache>`といった API ができるようです。

キャッシュという名前ですが、おそらく「キャッシュ用途に特化した組み込みステート管理システム」と見るのがよいでしょう。具体的な API の使い方は[このファイル](https://github.com/reactjs/server-components-demo/blob/859ba55c82c672b0965539f80879c5f55ad2a8f6/src/Cache.client.js)が比較的分かりやすいです。

キャッシュの中身は（まるで`useContext`のように）どこからでも取得できます。ただし React アプリ全体で保持されるものであり、Provider のような階層的構造はありません。キャッシュに対して React は refresh という操作をサポートしており、これは今あるキャッシュを破棄して再取得を要求します。ポイントは、このキャッシュシステムは Suspense との組み合わせを念頭に置いて作られた Suspense Cache だということです。Suspense Cache を活用するアーキテクチャでは、データの再取得（多くの場合これは非同期処理です）はコンポーネントのサスペンドを引き起こします。この際、複数のコンポーネントが同時にサスペンドしたとしても、それらのコンポーネントが同一のキャッシュを見るという一貫性のある挙動を React が保証してくれることになっています。以下に引用するように、この挙動に対する需要がわざわざ React 本体にこのようなシステムを組み込む理由となっています。

> Crucially, the cache object persists across multiple suspended render attempts. React will reuse the same cache (and therefore won't issue duplicate requests) on each attempt — even before the UI has mounted yet. **This is a key feature that isn't currently possible in a user space implementation.**
>
> [Built-in Suspense Cache](https://github.com/reactwg/react-18/discussions/25)から引用

また、`<Cache>`コンポーネントで Cache Boundary を定義することによって、ひとつのコンポーネントツリーの中で古いキャッシュと新しいキャッシュが混在することを許可できます。これにより、refresh の影響範囲を制御できます（よく見ると Refresh 用の API が`useCacheRefresh`というフック（コンポーネントに紐づいた機能）になっていて、その理由は`<Cache>`の存在です）。

上記の理由を除いたとしても、このように新旧混在が可能で、しかも Concurrent Rendering が発生しても壊れないキャッシュシステムをユーザーランドで作るのはたいへん難しく、それが React 本体にこれが導入される理由となっています。

> I like to think of the React cache as a lazily-populated, immutable snapshot of an external data source. Because React will always read from the snapshot, data in the external store can be mutated, changed, manipulated, deleted without affecting the current UI, or breaking the requirements of Concurrent React. React manages the lifetime of these snapshots, deduplicates requests, and provides high-level APIs for controlling when a new snapshot should be taken. In theory, this should shift implementation complexity away from the data framework and into React itself.
>
> [Built-in Suspense Cache](https://github.com/reactwg/react-18/discussions/25)から引用

なお、現在 React で非同期処理のキャッシュを行うためのものとして[SWR](https://swr.vercel.app/)などのライブラリが知られていますが、Built-in Suspence Cache はそういったライブラリを不要にするものではありません。むしろ、SWR などが裏で Built-in SUspence Cache を利用することで Concurrent Rendering に対応できるようになるものです。

## まとめ

この記事では React 18 alpha の発表をまとめて紹介しました。17 から 18 へのアップグレード自体は簡単とされていますが、その裏側をのぞいてみると思想の breaking change とでも言うべきものが結構あり、メジャーアップデートにふさわしい内容になっています。
