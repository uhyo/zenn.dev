---
title: "JavaScriptのasync/awaitはPromiseの糖衣構文か"
emoji: "🤝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "ecmascript"]
published: true
---

Q. **JavaScriptのasync/awaitはPromiseの糖衣構文か？**

A. 糖衣構文ではありません。

Q. **なぜ？**

A. `async function`構文で作られた関数オブジェクトに対して`Function.prototype.toString`メソッドを呼び出すと、`async function`で始まる文字列が得られます。これは`Promise`だけでは不可能で、`async function`構文を用いないと不可能です。

```js
async function foo() {}
// "async function foo() {}"
console.log(Function.prototype.toString.call(foo)); 
```

※ここでは糖衣構文は「その構文を使ったプログラムをその構文を使わない同じ意味のプログラムに常に書き換えられるような構文」と定義しています。

Q. **だから何？**

A. さあ……