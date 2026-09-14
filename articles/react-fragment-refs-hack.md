---
title: "Fragment Refsのずるい使い方"
emoji: "🪄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

**Fragment Refs**は、React 19.3で追加された新機能のひとつで、`Fragment`コンポーネントが`ref`をサポートするようになるというものです。

ここで得られる`ref`はReactが用意した`FragmentInstance`オブジェクトで、`addEventListener`や`focus`といったいくつかのメソッドを持っています。

`Fragment`は何もDOM要素を描画しない、実態のないラッパーですが、`FragmentInstance`を通じてそのラッパーに対して疑似的に操作を行っているかのように振る舞います。

基本的な使い方は[公式ドキュメント](https://react.dev/reference/react/Fragment#fragmentinstance)や他の記事に譲るとして、この記事では**ちょっとずるい使い方**を紹介します。

## 背景: Activityコンポーネント

`Activity`コンポーネントはReact 19.2で導入されたもので、`<Activity mode="hidden">`で囲まれた部分は非表示になります。

```jsx
<Activity mode="hidden">
  <h2>非表示セクション</h2>
  <p>この部分は非表示になります。</p>
</Activity>
```

`Activity`コンポーネントの利点はいくつかあります。

例えば、画面には見えないけど裏でレンダリングしておくことで、表示/非表示を素早く切り替えることができます。裏でレンダリングされる場合はレンダリングの優先度が下げられるため、パフォーマンスへの悪影響を抑えて準備できます。

また、非表示にしている間もコンポーネントの状態は保持されるため、再表示したときに初期化されることなくそのまま利用できます。

この`Activity`コンポーネントですが、それ自体は何もDOM要素を描画しません。その点では`Fragment`に似ています。そして、`mode="hidden"`の場合は中のDOM要素に対して`display: none`を与えることで、非表示という挙動を実現します。

```jsx
<Activity mode="hidden">
  <h2>非表示セクション</h2>  ← `display: none`が適用される
  <p>この部分は非表示になります。</p>  ← `display: none`が適用される
</Activity>
```

では、ここで問題です。

このように、「自分では何も描画せず、内部の要素に`display: none`を与える」という挙動のコンポーネントを自分で実装できるでしょうか？

ちょっと考えてみましょう。

実は、従来このようなコンポーネントを実装することはできませんでした。

しかし、Fragment Refsの機能を使うことで、ちょっと強引ですが、このような挙動を実現できるようになったのです。

## Hiddenコンポーネントを作る

では、`<Hidden enabled>...</Hidden>`のように使えるコンポーネントを、Fragment Refsを用いて作ってみましょう。

React 19.3の時点では、`FragmentInstance`が持つメソッドは以下のとおりです。

- addEventListener, removeEventListener
- dispatchEvent
- focus, focusLast, blur
- observeUsing, unobserveUsing
- getClientRects
- getRootNode
- compareDocumentPosition
- scrollIntoView

これらのうち、今回の要件で使えそうなのはどれでしょうか。

答えは`observeUsing`です。今回の要件では、Fragmentで囲まれたDOM要素を取得する必要があります。他のメソッドはDOM要素の取得には使えなそうです。

この`observeUsing`は、`IntersectionObserver`か`ResizeObserver`のインスタンスを与えることで、Fragmentの子として存在する各要素を監視することができます。

```ts
const observer = new IntersectionObserver(callback, options);
fragmentRef.current.observeUsing(observer);
```

この`observeUsing`は、実は各子要素に対して`observe`を呼んでくれるという挙動をします。この挙動を悪用して、`IntersectionObserver`でも`ResizeObserver`でもない独自のオブジェクトを渡すことで、`observe`を通じて子のDOM要素にアクセス可能になるのです。Fragmentに属するDOM要素が変化した場合でも、ちゃんと`unobserve`を呼んでくれます。

これを具体的な実装にするとこんな感じになります。以下の実装では、何となく`display: none`の代わりに`hidden`属性を与えるようにしています。

```tsx
function Hidden({ enabled, children }: { enabled: boolean; children: ReactNode }) {
  const attachObserver = useCallback(
    (instance: FragmentInstance) => {
      const observer = {
        observe(element: Element) {
          element.toggleAttribute('hidden', enabled)
        },
        unobserve(element: Element) {
          // Leave detached children clean in case they are moved elsewhere.
          element.removeAttribute('hidden')
        },
        // Never called by react-dom; only here so the object structurally
        // satisfies the ResizeObserver type that observeUsing() is declared
        // to accept.
        disconnect() {},
      }
      instance.observeUsing(observer)
      return () => {
        instance.unobserveUsing(observer)
      }
    },
    [enabled],
  )

  return <Fragment ref={attachObserver}>{children}</Fragment>
}
```

このように、`observeUsing`に与えた疑似Observerが、`observe`と`unobserve`メソッドを通じてFragmentの内部のDOM要素全てを取得できていることが分かります。もちろん、Fragmentの中に要素が追加された場合なども対応できます。

以下のページで実際に試すことができます。このページはFragment Refsのいろいろな使用例を集めたもので、「Hidden group」というタブでこの`Hidden`コンポーネントが動いています。

https://uhyo.github.io/react-193-playground/

## 注意

公式ドキュメントでは`observeUsing`は`IntersectionObserver`か`ResizeObserver`のインスタンスを渡すことを想定していると記載されています。そのため、この記事のやり方はずるいどころか、公式にサポートされたものではありません。将来動かなくならない保証はありませんので、実際のプロダクトで使うのはやめたほうがよいでしょう。

また、上記の実装は単純化されており、子のDOM要素がもともと`hidden`属性を持っていた場合を考慮していません。必要に応じて改良してみてください。

## まとめ

この記事では、ReactのFragment Refsを使ったちょっとずるい方法で、内部のDOM要素に`hidden`を与える`Hidden`コンポーネントを実装する方法を紹介しました。

もともとActivityコンポーネントが子のDOM要素に干渉する特殊な挙動を持っていてずるいと思い、ユーザーランドでも同じことができるようになるのを期待していました。

そのためFragment Refsが出たときはこれだと思ったのですが、実際にやってみると思いのほかずるい感じになってしまいました。公式にこのユースケースがサポートされると嬉しいですね。