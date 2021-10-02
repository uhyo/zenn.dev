---
title: "TypeScript 4.5でますます便利に！ better-typescript-lib v2"
emoji: "🤓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

今日リリースされた [TypeScript 4.5 Beta](https://devblogs.microsoft.com/typescript/announcing-typescript-4-5-beta/) の新機能として、**標準ライブラリの差し替えが従来よりも簡単になる**というものがあります。

筆者は TypeScript の標準ライブラリから`any`を排除してより安全にした[better-typescript-lib](https://github.com/uhyo/better-typescript-lib)を開発していましたが、このたび TypeScript 4.5 に対応した v2.0.0 のベータ版を用意しました（`2.0.0-beta`）。

この記事では better-typescript-lib の簡単な紹介に加えて、TypeScript 4.5 の機能の解説やそれによって better-typescript-lib に起こった変化を紹介します。

:::message
英語版記事も用意しました。合わせてご覧ください。

→ [TypeScript 4.5 Shortens Path to Safer Standard Library  - DEV Community 👩‍💻👨‍💻](https://dev.to/uhyo_/typescript-4-5-shortens-path-to-safer-standard-library-256m)
:::

## better-typescript-lib について

better-typescript-lib は、TypeScript の標準ライブラリをより型安全にしたものです。better-typescript-libでより安全になる例として、`JSON.parse`や`eval`が挙げられます。標準の TypeScript ではこれらの返り値は`any`になります。

```ts
// any 😩
const obj = JSON.parse('{"foo": 123}');

console.log(obj.foo); // 123
```

better-typescript-lib を導入すると、`JSON.parse`の返り値は`JSONValue`に、`eval`の返り値は`unknown`になります。`JSONValue`は JSON として表現されうる値全てのユニオン型です。こうすると、`any`型ではないので少し取り扱いが面倒になりますが、その代わりに型安全になります。

```ts
// JSONValue 😁
const obj = JSON.parse('{"foo": 123}');

if (isFooObject(obj)) {
  console.log(obj.foo); // 123
}

function isFooObject(obj: JSONValue): obj is { foo: number } {
  return isPropertyAccessible(obj) && typeof obj.foo === "number";
}
function isPropertyAccessible(obj: unknown): obj is Record<string, unknown> {
  return obj !== null;
}
```

このように書かれるとかなり面倒な印象を受けるかもしれませんが、これは本来やらなければいけないことです。`any`を排除することによって、やらなければいけないことにしっかりと気づくことができます。better-typescript-lib を使用することで、これまで`any`によって隠されていたミスを発見できるのです。

詳しくは以前に執筆したこちらの記事もご覧ください。

https://qiita.com/uhyo/items/18458646e8aae25207db

## 従来の better-typescript-lib 導入方法

このように便利な better-typescript-lib ですが、従来は導入がやや大変でした。TypeScript が用意してくれる標準ライブラリの読み込みを無効化して、better-typescript-lib を手動で読み込む必要がありました。

具体的には、`npm i -D better-typescript-lib`し、`tsconfig.json`に`"noLib": true`と書き、さらにアプリケーションコードの中から次のように読み込む必要がありました。これはあまり美しくありませんね。ハック感もすごく出ています。

```ts
/// <reference path="./node_modules/better-typescript-lib/lib.es5.d.ts" />
/// <reference path="./node_modules/better-typescript-lib/lib.dom.d.ts" />
```

## 新しい better-typescript-lib 導入方法とその仕組み

TypeScript 4.5 以上対応の better-typescript-lib v2 では、better-typescript-lib の導入は **npm install するだけ**になります。つまり、次を実行するだけであり、手動での読み込みはおろか、tsconfig.json の書き換えすら不要です。非常に簡単ですね。

```
npm i -D better-typescript-lib@2.0.0-beta
```

:::message
TypeScript 4.5の正式版がリリースされたらそれに合わせてbetter-typescript-libもv2を正式リリースする予定です。その後は`@2.0.0-beta`は不要です。最新情報は[GitHub](https://github.com/uhyo/better-typescript-lib)をご確認ください。
:::

これを可能にしているのが、TypeScript 4.5 で導入される次の機能です。

https://github.com/microsoft/TypeScript/pull/45771

これは、`@typescript/lib-[lib]`という名前のパッケージがインストールされている場合、それを標準ライブラリの`lib.[lib].d.ts`の代わりに読み込むというものです。例えば tsconfig.json に`"target": "es2015"`と書かれている場合は TypeScript に付属の`lib.es2015.d.ts`が読み込まれるというのがこれまでの挙動ですが、もし`@typescript/lib-es2015`がインストールされている場合、こちらが代わりに読み込まれます。

これにより、改善された型定義を`@typescript/lib-es2015`のような名前でインストールしてやることで、TypeScript が読み込む型定義を差し替えることができます。

なお、npm で`@typescript`というスコープを所持しているのはもちろん TypeScript チームです。第三者が勝手にこれらのパッケージ名を publish することはできません。しかし幸いにも、npm（あるいは package.json をサポートする他のパッケージマネージャも）はパッケージを別名でインストールする機能をサポートしています。`better-typescript-lib`の`package.json`は次のような内容になっています。

```json
  "dependencies": {
    "@typescript/lib-dom": "npm:@better-typescript-lib/dom@2.0.0-beta",
    "@typescript/lib-es2015": "npm:@better-typescript-lib/es2015@2.0.0-beta",
    "@typescript/lib-es2016": "npm:@better-typescript-lib/es2016@2.0.0-beta",
    "@typescript/lib-es2017": "npm:@better-typescript-lib/es2017@2.0.0-beta",
    "@typescript/lib-es2018": "npm:@better-typescript-lib/es2018@2.0.0-beta",
    "@typescript/lib-es2019": "npm:@better-typescript-lib/es2019@2.0.0-beta",
    "@typescript/lib-es2020": "npm:@better-typescript-lib/es2020@2.0.0-beta",
    "@typescript/lib-es2021": "npm:@better-typescript-lib/es2021@2.0.0-beta",
    "@typescript/lib-es5": "npm:@better-typescript-lib/es5@2.0.0-beta",
    "@typescript/lib-esnext": "npm:@better-typescript-lib/esnext@2.0.0-beta",
    "@typescript/lib-header": "npm:@better-typescript-lib/header@2.0.0-beta",
    "@typescript/lib-scripthost": "npm:@better-typescript-lib/scripthost@2.0.0-beta",
    "@typescript/lib-webworker": "npm:@better-typescript-lib/webworker@2.0.0-beta"
  }
```

これは例えば、`@typescript/lib-dom`という名前で`@better-typescript-lib/dom@2.0.0-beta`をインストールするというような意味です。こうすることで、好きなパッケージを`@typescript/lib-[lib]`という名前でインストールすることができます。これを利用して、我々が用意した型定義を TypeScript に読み込ませることができます（これもややハックな気がしますが、TypeScript が公式に推奨している方法です）。

`better-typescript-lib`をインストールするだけでこれらの`@typescript/lib-[lib]`が TypeScript コンパイラから見えるようになるのは、node_modules の中身がフラット化される仕様を使用しているからです。PnP などの場合はうまくいかないかもしれませんが、要望次第で今後の対応が検討されるかもしれません。

ちなみに、TypeScript にこのような機能が導入されたのは、もともとは`@types/web`の構想のためと思われます（次の issue を参照）。

https://github.com/microsoft/TypeScript/issues/44795

TypeScript の標準ライブラリは TypeScript のバージョンアップに合わせて変化しますが、その中でも`lib.dom.d.ts`の変更は後方互換性が壊れる原因になりがちでした。`@types/web`を`@typescript/lib-dom`としてインストールしておくことで、独立したバージョン管理を可能として TypeScript のバージョンアップの影響を受けないようにすることができます。これを実現するための方式はいろいろありますが、現在の`@typescript/lib-[lib]`を使う方式は、依存先のバージョン管理などを既存のパッケージマネージャに完全に任せることができるという利点のために選択されました。`better-typescript-lib`のインストールの簡単さもまさにこの恩恵を受けていますね。

## まとめ

TypeScript 4.5 では標準ライブラリの型定義を簡単に差し替えるための新たな機能が導入されました。デフォルトのものより型安全な標準ライブラリを提供する`better-typescript-lib`では、次期メジャーバージョンでさっそくこの機能を利用し、より簡単にインストールできるようになりました。

- [better-typescript-libのGitHub](https://github.com/uhyo/better-typescript-lib) （v2の正式リリース前は`v2`ブランチを参照してください）