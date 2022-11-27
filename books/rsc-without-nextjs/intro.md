---
title: "はじめに"
---

最近Next.js 13がリリースされ、`app`ディレクトリのサポートがベータ版として追加されました。`app`ディレクトリは数年前から情報が公開されていた**React Server Components** (**RSC**) を基盤として作られており、（ベータ版とはいえ）一般のReactユーザーの手の届くところにReact Server Componentsがやってきたことになります。

RSCはあくまでReactの機能であり、Next.jsの専売特許ではありません。例えばGatsbyなどもRSCのサポートを進めています。しかし、ReactのサーバーサイドレンダリングにおいてNext.jsがデファクトスタンダードとなっている現状では（最近Remixもがんばっていますが）、RSCの話題にはどうしてもNext.jsが付いてきます。

そこで、この本では基礎からReact Server Componentsを理解し直すことを目的として、Next.jsに頼らずにRSCを使ってみるハンズオンを提供します。RSCは色々使い道がありますが、筆者はRSCのひとつの側面として「（従来の）Reactアプリとうまく統合された便利なテンプレートエンジン」というものがあると考えています。そこで、この本ではRSCを事前にプリレンダリングすることを目標とします。Next.jsで言うところのStatic Generation的なユースケースを自前でやってみようということです。

この本に出てくるコードは以下のリポジトリで閲覧することができます。

https://github.com/uhyo/rsc-without-nextjs