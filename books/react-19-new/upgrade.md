---
title: "React 19へのアップグレード"
---

React 19は、メジャーバージョンということで破壊的変更も含んでいます。これについては、[公式のアップグレードガイド](https://react.dev/blog/2024/04/25/react-19-upgrade-guide)があるのでそれを参照するのがよいでしょう。

折角なので、この記事でも概要とインパクトについてかいつまんでお伝えします。

## `propTypes`と`defaultProps`の一部廃止

`propTypes`は、コンポーネントのpropsの型を指定してランタイムにチェックしてくれる仕組みです。

```jsx
import PropTypes from 'prop-types';

const MyComponent = ({ name }) => {
  return <div>Hello, {name}</div>;
};

MyComponent.propTypes = {
  name: PropTypes.string.isRequired,
};
```

今どきはこういうのはTypeScriptなどの型システムでチェックできるので、`propTypes`は廃止されました。指定してもエラーにはなりませんが何も起きません。

`defaultProps`についても、ES2015より前の時代に需要があったものとして、関数コンポーネントについては廃止されました。一方、クラスコンポーネントについては簡単な代替方法がないため引き続き使えます。

`defaultProps`については、使用していた場合はランタイムの挙動の変化が伴うため、React 19にアップグレードする前に対応が必要です。これについてはcodemodも無いので頑張って対応しましょう。とはいえ、`defaultProps`を使っているのは相当古いコードベースでしょうから、それをReact 19までアップグレードするやる気があればきっと成し遂げられるでしょう。

## Legacy Contextの廃止

現在ReactのContextといえば`createContext`で作って`useContext`で呼び出すものを指しますが、それより前にもContextの仕組みがありました。それがLegacy Contextです。これは新しいContextの仕組みが登場したことで非推奨となり、ついにReact 19で廃止されます。

ちなみに、もともとクラスコンポーネントでしかサポートされていませんでした。

## string refの廃止

string refは、`ref`にrefオブジェクトや関数ではなく文字列を渡せるもので、クラスコンポーネント限定の機能でした。

```jsx
class MyComponent extends React.Component {
  componentDidMount() {
    this.refs.myInput.focus();
  }

  render() {
    return <input ref="myInput" />;
  }
}
```

refコールバックで普通に代替できるし、refコールバックなどに比べて表現力に乏しいので廃止されました。パフォーマンス面でも良くなかったらしいです。

## ReactDOM.renderの廃止

`ReactDOM.render`は、React 18で`createRoot`に置き換えられました。React 18では新旧どちらも使えましたが、React 19で`ReactDOM.render`が廃止されることになります。

```jsx
// 古い書き方
ReactDOM.render(<App />, document.getElementById('root'));

// 新しい書き方
ReactDOM.createRoot(document.getElementById('root')).render(<App />);
```

Reactのサンプルコードを手書きするときなど、うっかり`render`と書いてしまって古い人間であることを露呈してしまわないように気を付けましょう。

ちなみに、`ReactDOM.hydrate`も同時に廃止されます。

## `useRef`に引数が必須になった

正確にはこれはReactの破壊的変更ではなく`@types/react`の型定義の破壊的変更になりますが、`useRef`に引数が必須になりました。

```tsx
// これまでOKだったコード
const ref = useRef<HTMLDivElement>();

// 修正後
const ref = useRef<HTMLDivElement>(undefined);
```

ランタイムにおいては渡されなかった引数は`undefined`となるのでこの変更はランタイム的には安全ですが、影響箇所は大きそうですね。この変更はcodemodが提供されています。

ちなみに、`useRef`の返り値の型が場合によって`RefObject`と`MutableRef`の2種類があることにお気づきだった方も多いでしょう。React 19向けの型定義では`MutableRef`はdeprecatedとなり、`RefObject`のみが使われるようになりました。

## `useReducer`のシグネチャが変更

これも型定義の話ですが、`useReducer`の型引数が変更されました。

```tsx
// これまでの型
useReducer<React.Reducer<State, Action>>(reducer, initialState);

// 新しい型
useReducer<State, [Action]>(reducer, initialState);
```

従来は関数全体の型を型引数として指定していたところ、新しいシグネチャではステートの型とアクションの型が別々になりました。前の定義にどんな問題があったのか具体的には知りませんが、一般的には新しいシグネチャのほうが筋が良さそうに思います。

この変更により、`useReducer`に明示的に型変数を渡していたところは壊れることになります。

しかし、もともと`useReducer`に型定義を渡す必要が合った場面は少ないはずです。`reducer`関数の型定義がちゃんとしていればうまく推論されるからです。修正自体も、codemodは無いとはいえ、型だけの話なのでリスクは低いでしょう。