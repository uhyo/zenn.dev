---
title: "はじめに"
---

:::message
2023/10/29: バージョンアップしてReact Canaryを用いるようにしました！
:::

最近Next.js 13がリリースされ、`app`ディレクトリのサポートがベータ版として追加されました。`app`ディレクトリは数年前から情報が公開されていた**React Server Components** (**RSC**) を基盤として作られており、（ベータ版とはいえ）一般のReactユーザーの手の届くところにReact Server Componentsがやってきたことになります。

RSCはあくまでReactの機能であり、Next.jsの専売特許ではありません。例えばGatsbyなどもRSCのサポートを進めています。しかし、ReactのサーバーサイドレンダリングにおいてNext.jsがデファクトスタンダードとなっている現状では（最近Remixもがんばっていますが）、RSCの話題にはどうしてもNext.jsが付いてきます。

そこで、この本では基礎からReact Server Componentsを理解し直すことを目的として、Next.jsやその他のフレームワークに頼らずに、ReactのAPIを直接利用してRSCを使ってみるハンズオンを提供します。というよりは、筆者がRSCを触ってみたメモを本にまとめたような感じになっています。

この本に出てくるコードは以下のリポジトリで閲覧することができます。

https://github.com/uhyo/rsc-without-nextjs

**ご注意**: この本の内容はまだstableに達していない機能に対する解説を含んでいます。この本の内容はこの本が書かれた当時のものであり、その後変化している可能性があります。