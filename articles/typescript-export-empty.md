---
title: "export {}; が使われるTypeScript特有の事情"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ecmascript", "typescript"]
published: true
---

TypeScriptのコードでは、`export {};` という記述を見かけることがあります。これはECMAScriptの構文ではあるものの、これが使われる背景にはTypeScript特有の事情があります。この記事では、`export {};` がなぜ使われるのか、どのような効果があるのかを解説します。

## `export {};` とは

この構文は、`export`というキーワードから分かるように、モジュールに関連する構文です。

一般に、`export { ... };`という構文は、既存の変数をモジュールからエクスポートするために使われます。例えば、次のようなコードが考えられます。

```ts
const foo = 42;
const bar = "hello";
const banana = "banana";

export {
  foo,
  bar as hello,
  banana as "🍌",
};
```

変数をエクスポートする場合は`export const foo = 42;`のように書くこともできますが、`export`を付けずに宣言された既存の変数をエクスポートする場合は、`export { ... };`を使います。この構文は、複数の変数を一度にエクスポートする場合に便利なほか、`as`キーワードを使ってエクスポート名を変更することもできます。

さて、`export {};`もこの構文の一種です。つまり、0個の変数をエクスポートするという意味になっています。

```ts
export {};
```

0個の変数をエクスポートするというのは、何もエクスポートしないということです。つまり、この`export`宣言はあっても無くても変わりません。ECMAScript的には書く意味が無いことになります。

## TypeScriptで`export {};`が使われる理由

以上の説明から、`export {};`は何もしない構文であることが分かりました。では、なぜTypeScriptのコードでこの構文が使われるのでしょうか。ここにはTypeScript特有の事情があります。実は、その事情はTypeScriptにおける**モジュール**の判定に関係しています。

### ECMAScriptにおけるスクリプトとモジュール

ECMAScriptには、スクリプトとモジュールという2つの概念があります。スクリプトは従来のコードで、モジュールはES2015から導入されました。モジュールの特徴は`import`と`export`が使えることです。一方、スクリプトは`import`と`export`が使えません（他にもモジュールは常にstrict modeで実行されるなどの違いがあります）。

あるJavaScriptコードがスクリプトなのかモジュールなのかの判別方法は、実は仕様で定められていません。そのコードがスクリプトなのかモジュールなのかは、そのコードを実行する環境に依存します。

例えば、HTML（ブラウザ上）では`<script>`で読み込まれたコードはスクリプトとして扱われ、`<script type="module">`で読み込まれたコードはモジュールとして扱われます。もし、`<script>`で読み込まれたコードに`import`や`export`が含まれていた場合、そのコードは構文エラーとなります。

また、Node.jsの場合はファイルの拡張子が`.cjs`であればスクリプト、`.mjs`であればモジュールとして扱われます。

### TypeScriptにおけるモジュールの判定

TypeScriptは独自にモジュールとスクリプトの判定を行います。問題となるのは、基本的にTypeScriptコンパイラからはそのコードがどのように使われるのかは分からないということです。そのため、実際の実行環境のように、使われ方によってスクリプトかモジュールかを判定することはできません。

TypeScriptは、コード中で`import`や`export`が使われているかどうかで、そのコードをモジュールとして扱うかスクリプトとして扱うかを判定します。もし、`import`や`export`が使われていない場合、そのコードはスクリプトとして扱われます。

これは、普通の実行環境がコードをパースする前にもうモジュールかスクリプトかを判定しているのと比べて特殊です。TypeScriptでは、コードをパースしてからそのコードがモジュールかスクリプトかを判定します。

### `export {};`の効果

これで、`export {};`がどのような効果を持つのかが分かりますね。何も`import`も`export`もしないけどモジュールとして扱いたい場合、`export {};`を使うことで、TypeScriptコンパイラにそのコードをモジュールとして扱うように指示することができるのです。

TypeScript以外ではコードを見る前にモジュールかスクリプトかを判定するので、`export {};`を書く必要はありません。その点で、このような`export {};`の使い方はTypeScriptに特有のものと言えます。

### なぜコードをモジュールとして扱いたいのか

ところで、インポートもエクスポートもしないファイルをモジュールとして扱いたいのはどうしてでしょうか。

これは、スクリプトとモジュールの違いによるものです。特に、スクリプトにおいて`var`で宣言された変数はグローバルスコープになりますが、モジュールではモジュールスコープになります。

```html
<script>
  var foo = 42;
</script>
<script type="module">
  var bar = 42;
</script>
<script>
  console.log(foo); // 42
  console.log(bar); // ReferenceError: bar is not defined
</script>
```

TypeScriptもこの挙動をサポートしています。特に、スクリプト内で定義された変数や型はグローバルな定義となり、プロジェクト内の他のファイルからも参照できます。逆に、モジュール内で定義された変数や型は、そのモジュール内でのみ参照できます（他のモジュールから参照するためにはエクスポートする必要があります）。

今どきのプロジェクトは、モジュールを使ってコードを書くことが一般的です。しかし、TypeScriptにおいては敢えてスクリプトとしたファイルに型定義を書くことでグローバルな型定義を用意する手法もあります。TypeScript初心者の方がはまりがちな罠として、そのようなファイルでうっかり`import`を書いてしまい、モジュールになってしまって他のファイルから型定義が参照できなくなってしまうことがあります（ちなみに、そのような場合は`declare global`を使えばモジュールの中からグローバルな型定義を書くこともできます）。

逆に、スクリプトとして扱ってほしくないファイルに`export {};`を書くことで、そのファイルをモジュールとして扱うことができます。これにより、そのファイル内の定義がグローバルなスコープに漏れることを防ぐことができます。

実際のコードではモジュールは何かしらをエクスポートすることが多く、`export {};`を使うことはあまりないかもしれません。しかし、開発途中でまだエクスポートするものがない場合などに一時的に`export {};`を使うことがあるかもしれません。

### TypeScriptの型定義における具体例

TypeScriptの型定義ファイル（`.d.ts`）においても、スクリプトかモジュールかという区別は以上の説明がそのまま当てはまります。実は、`export {};`のテクニックがTypeScriptの標準ライブラリ（組み込みの型定義）でも使われています。

TypeScript 5.6 Beta向けに書かれた `lib.esnext.iterator.d.ts` という[型定義ファイル](https://github.com/microsoft/TypeScript/pull/58222/files)を見てみましょう。

```ts:lib.esnext.iterator.d.tsから抜粋
// NOTE: This is specified as what is essentially an unreachable module. All actual global declarations can be found
//       in the `declare global` section, below. This is necessary as there is currently no way to declare an `abstract`
//       member without declaring a `class`, but declaring `class Iterator<T>` globally would conflict with TypeScript's
//       general purpose `Iterator<T>` interface.
export {};

// Abstract type that allows us to mark `next` as `abstract`
declare abstract class Iterator<T> { // eslint-disable-line @typescript-eslint/no-unsafe-declaration-merging
    abstract next(value?: unknown): IteratorResult<T, undefined>;
}

// Merge all members of `BuiltinIterator<T>` into `Iterator<T>`
interface Iterator<T> extends globalThis.BuiltinIterator<T, undefined, unknown> {}

// Capture the `Iterator` constructor in a type we can use in the `extends` clause of `IteratorConstructor`.
type BuiltinIteratorConstructor = typeof Iterator;

declare global {
    // Global `BuiltinIterator<T>` interface that can be augmented by polyfills
    interface BuiltinIterator<T, TReturn, TNext> {
// 以下略
```

このファイルでは前述の`declare global`が使われています。これまでTypeScriptの組み込み型定義は基本的にスクリプトとして書かれていたため、ファイル内に書かれた定義が全て自動的にプロジェクト全体で利用可能でした。しかし、このファイルでは″内部実装”を`export {};`で隠蔽し、外に公開する型定義だけを`declare global`で書いています。

標準ライブラリに限らず、このようなパターンはプロジェクト内で共有される型定義を書くときにも使えるテクニックです。

## 余談: `moduleDetection`コンパイラオプション

実は、ここまで説明した内容は今でも有効ではあるものの、少し古くなりつつあります。それは、TypeScript 4.7で導入された`moduleDetection`コンパイラオプションにより、スクリプトかモジュール化の判定方法が変わるためです。

このオプションは以下の3つの値を取ります。

- `auto`（デフォルト）: コード内の`import`や`export`の有無に基づいてスクリプトかモジュールかを判定する。しかし、`module`オプションが`node16`または`nodenext`の場合、拡張子（`.mts`）や`package.json`の`type`フィールドの設定に従う[^moduleDetection_auto]。
- `legacy`: コード内の`import`や`export`の有無のみを見る（TS 4.6以前と同じ）。
- `force`: 全てモジュールとして扱う。

[^moduleDetection_auto]: 他に`.tsx`ファイルの取り扱いの変化などもありますが省略。

TS 4.7は`module`: `node16`などのNode.jsのESMサポートが追加されたバージョンです。それにより、従来TypeScriptでは内的要因（ファイルの中身）によってスクリプトかモジュールかを判定していたところ、Node.jsと同じような外的要因（拡張子や`package.json`の`type`フィールド）によって判定する仕組みも追加されました。

`force`はもはやスクリプトという概念を消し去り、全てのファイルがモジュールになります。今どきのTypeScriptプロジェクトはモジュールしか使わないからという理由の他にも、[逆に独立したファイルの寄せ集めのプロジェクトで勝手にグローバルスコープになってほしくない](https://github.com/microsoft/TypeScript/issues/14279)という需要もあったようです。

ただし、`auto`や`force`の挙動は型定義ファイル（`.d.ts`など）には当てはまらないことに注意してください。型定義ファイルは従来通りのルール（`import`/`export`があるか）で判定されるので、`export {};`のテクニックは有効です。上で紹介した標準ライブラリの例も型定義ファイルですね。

このように普通のファイルと型定義ファイルでルールが分かれるのは、スクリプト内で定義された変数がグローバル変数になっても嬉しい場面はとくに無い一方、型定義ファイル内で定義された型をプロジェクト全体で利用するというニーズはあるからなのでしょう。

## まとめ

TypeScriptのコードで`export {};`という構文を見かけることがありますが、これは何もエクスポートしないという意味です。TypeScriptのコードをモジュールとして扱いたい場合に使われるテクニックで、特に型定義ファイルなどで使われることがあります。


