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

A. `async function`構文で作られた関数オブジェクトに対して`Function.prototype.toString`メソッドを呼び出すと、`async function`を返す文字列が得られます。これは`Promise`だけでは不可能で、`async function`構文を用いないと不可能です。

```js
async function foo() {}
// "async function foo() {}"
console.log(Function.prototype.toString.call(foo)); 
```

Q. **だから何？**

A. さあ……