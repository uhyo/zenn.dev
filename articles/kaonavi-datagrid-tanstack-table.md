---
title: "DataGridを実装して感じたTanStack Tableに対する所感"
emoji: "📊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "tanstacktable"]
published: true
---

皆さんこんにちは。この記事は[株式会社カオナビ Advent Calendar 2024](https://qiita.com/advent-calendar/2024/kaonavi)の5日目の記事です。

今回は、筆者が弊社のデザインシステムである[sugao](https://brand.kaonavi.jp/sugao)に新しいコンポーネントを実装したときの話です。

sugaoは、デザインシステムあるあるとして、デザインの定義であると同時に再利用可能なコンポーネント集でもあります。ということで、必要に応じてコンポーネントの追加や修正も行われています。

今回取り上げる**DataGrid**というコンポーネントは、[MUIのDataGrid](https://mui.com/x/react-data-grid/)に近いものだと思っていただくと良いでしょう。つまり、複雑なテーブルを表示するためのコンポーネントです。

![DataGridの表示例。見出し行と、2行2列からなるテーブルが表示されている。](/images/kaonavi-datagrid-tanstack-table/datagrid.png)


筆者は、DataGridコンポーネントの内部実装に **[TanStack Table](https://tanstack.com/table/latest)** を採用しました。これは、フロントエンドで使えるライブラリ集であるTanStackに属する、テーブルライブラリです。ということで、今回はDataGridの実装を通してTanStack Tableに向き合う機会がありましたので、その所感をお伝えします。

## 結論

TanStack Tableは別に無くてもいける。

## 実装の方針

では、ここからDataGridの実装時に考えていたことを見ていきます。

まず、TanStack Tableはヘッドレスなライブラリです。つまり、TanStack Table自体にはCSS関連の実装は含まれておらず、TanStack Tableを使う側でスタイリングを実装する必要があります。逆に、それ以外の部分はTanStack Tableに任せることができます。

そのため、DataGrid = TanStack Table + CSS という考え方ができます。筆者もその考え方で実装を進めました。

### 腐敗防止層の考え方

このようにサードパーティーライブラリに依存するときに重要な考え方が、**腐敗防止層**です。ここでは、アプリケーションコードから直にサードパーティライブラリに依存するのではなく、間に自家製のラッパーを噛ませることで、サードパーティライブラリの変化に強くする設計手法を指します。

腐敗防止層についてはこのスライドも参考になります。

https://speakerdeck.com/okashoi/interface-and-anti-corruption-layer

この記事におけるDataGridは、TanStack Tableに対する腐敗防止層としての役割も期待されています。

DataGridはReactコンポーネントとして実装したのでReactの語彙を使いますが、このときに重要なのが、**TanStack Tableの型を直接DataGridのpropsとして露出しない**ということです。

TanStack Tableの型をそのままDataGridのpropsとして露出してしまうと、TanStack Tableに破壊的な変更が発生した場合、もしくはTanStack Tableではなく別の何かに乗り換えたくなった場合に、DataGridの利用者側にまで影響が及んでしまいます。まさにそれを防ぐのが腐敗防止層の役割です。

ということで、DataGridのprops設計は、「TanStack Tableの型定義は忘れて独自にDataGridのpropsを定義する。DataGrid内でTanStack Tableが受け付ける型に変換する」というやり方で行いました。

## DataGridのprops設計

実際のコードよりかなり簡略化していますが、普通にやると、propsはおおよそこのような感じになります。

```ts
type DataGridProps<Data, CustomState> = {
  // テーブルの内容を、データの配列として与える
  data: readonly Data[];
  // 列の定義
  columns: readonly DataGridColumnDefinition<Data, CustomState>[];
  // 実際にはCustomStateを使わない場合は指定しなくてもいい定義になっています
  state: CustomState;
};
```

DataGridのようなコンポーネントは、このように`data`と`columns`を受け取るインターフェースが定番です。これにより、ユーザー側がループを回して`<tr><td>...`みたいなマークアップを手書きする必要がなくなります。この辺りの実装を実装詳細としてコンポーネント内に隠すのもDataGridの役割のひとつです。

また、`state`を別に受け取るようにしているのもポイントです。例えば「テーブル内にチェックボックスがレンダリングされておりチェックできる」のような実装をした場合に`state`が役に立ちます。

チェックボックスの状態は`data`に組み込んでもいいのですが、そうすると状態の変化のたびに`data`が新しいオブジェクトとなり、メモ化したい場合に不都合があります。頻繁に変化するステートは`data`から切り出して`state`とすることによって、テーブルの再描画の高速化をしやすくなります。

この`data`, `columns`, `state`のような分担はTanStack Tableにも同じ概念があるので、ガワだけ変えている感じです。

### 列の定義

`columns`で列の定義を与えることで、実際にDataGridのテーブル内にレンダリングされる内容が決まります。

DataGridはカスタマイズ性の高いテーブルコンポーネントとするために、各セルの内容は自由に決めることができます。

そのため、1つの列の定義（`DataGridColumnDefinition<Data, CustomState>`）は大体こんな感じになります（例によって、実物より簡略化されています）。

```ts
type DataGridColumnDefinition<Data, CustomState> = {
  /**
   * ヘッダーをレンダリングする関数
   */
  renderHeader: (
    state: CustomState
  ) => CellRendering;
  /**
   * セルをレンダリングする関数
   */
  renderCell: (data: Data, state: CustomState, index: number) => CellRendering;
};

export type CellRendering =
  | React.ReactChild
  | null
  | {
      content: React.ReactNode;
      // CellOptionsはセルの背景色とかの設定ができるオブジェクトだが省略
      options?: CellOptions;
    };
```

要するに、`renderCell`は`(data) => <span>{data.text}</span>`みたいな感じで自由にJSXを描画できるAPIになっているということです。ヘッダー部分を定義する`renderHeader`もあります。

これは自然な設計のように思えますね。しかし、ここですでにTanStack Tableの考え方と乖離が発生しています。

実は、TanStack Tableでは列は次の3種類に分類されます。

- **Accessor Columns**（アクセサ列）: 元データ（`Data`）から数値や文字列などを取り出し、それをセルに紐付けて表示する列
- **Display Columns**（ディスプレイ列）: `Data`をもとに自由に表示内容を決められる列
- **Grouping Columns**（グルーピング列）: 複数の列をグルーピングするための列

上のような`renderCell`のようなAPIは、すべて**Display Columns**に対応するものです。言い換えれば、DataGridでは、Accessor Columnsは利用しない決定をしたということです。

### アクセサ列を使わない

アクセサ列はTanStack Tableの一部機能を利用するために必要です。

具体的には、[行の並び替え](https://tanstack.com/table/latest/docs/api/features/sorting)や[フィルタリング](https://tanstack.com/table/latest/docs/api/features/global-filtering)などは、アクセサ列が必要な機能です。

これらは、TanStack Tableに渡した`data`を、TanStack Table内部でソートしたりフィルタリングしてから表示してくれる機能です。

今回、これらの機能はDataGridの機能には不要で、DataGridに渡す`data`自体を事前にソートやフィルタリングしておけばいいだろうと判断しました。理由としては、実装のやり方がむやみに増えてしまうことや、`renderCell`のAPIのシンプルさを優先したことが挙げられます。

他にも、列の選択状態や開閉状態を保存してくれる機能などもありますが、これらは`state`に保存されるので、TanStack Tableにロックインされずとも同等の自前実装が可能です。現在のところ、これらはDataGridに敢えて組み込むまでもないと考え自前実装しています。

なお、実際のDataGridには列のグルーピング機能が存在しているので、TanStack Tableのグルーピング列は使用しています。

## TanStack Tableでやってくれないこと

TanStack Tableはヘッドレスなライブラリなので、UI周りはやってくれません。そのため、表示周りのカスタマイズ機能をDataGridに追加する場合は自前の実装となります。

### 罫線のカスタマイズ

その一つは、**罫線のカスタマイズ機能**です。DataGridでは、列の定義に`border: 'left'`と書けばその列の左に線が引かれるようなインターフェースとなっています。太さや色のカスタマイズも可能で、`border: 'both'`で列の行側に線を引くことも可能です。

罫線は列と列の間、あるいは行と行の間に引かれるものですから、1つの線に対して関係者が2人いることになります。そこの処理をうまく定義するといったことは自前で行う必要があります。

![Figma Communityに公開されたsugaoのデザインの「DataGrid」部分のスクリーンショット。さまざまなカスタマイズをされた罫線の表示パターン。](/images/kaonavi-datagrid-tanstack-table/borders.png)

### 列の結合

複雑なテーブルにおいては、列を結合して表示することの需要があります。

DataGridでは、「列グループ」を定義して、列グループに対して`renderCell`が定義されている場合は、複数列を結合してその結果が反映されるというAPIにしました。具体的には、このようなインターフェースになっています。列グループの`renderCell`が無い（または`false`）を返した場合は、その列については列の結合が発生せず、個別の列の`renderCell`が利用されます。

```ts
/**
 * 列グループの定義
 */
type DataGridGroupColumnDefinition<Data, CustomState> = {
  group: true;
  columns: readonly DataGridColumnDefinition<Data, CustomState, true>[];
  /**
   * ヘッダーをレンダリングする関数。
   * この関数でレンダリングされたヘッダーは列グループ全体にまたがって（横結合）表示される。
   * falseを返すことで、列グループのヘッダーを描画せずにグループ内の列に任せることができる。
   * renderHeaderを指定しなかった場合もグループ内の列に任せる。
   */
  renderHeader?: (state: CustomState) => CellRendering | false;
  /**
   * セルをレンダリングする関数。
   * この関数でレンダリングされたセルは列グループ全体にまたがって（横結合）表示される。
   * falseを返すことで、列グループのセルを描画せずにグループ内の列に任せることができる。
   * renderCellを指定しなかった場合もグループ内の列に任せる。
   */
  renderCell?: (
    data: Data,
    state: CustomState,
    index: number
  ) => CellRendering | false;
};
```

この機能を用いると、画像のような表示が可能になります。

![Figma Communityに公開されたsugaoのデザインの「DataGrid」部分のスクリーンショット。セル結合の表示パターン。](/images/kaonavi-datagrid-tanstack-table/span.png)



API的には結構きれいに抽象化できていると思うのですが、TanStack Tableは実はこの辺りの面倒を見てくれません。

ヘッダー行が画像のように多段になっていることや、ヘッダー部分の横結合はTanStack Tableで表現できるのですが、セル部分の横結合は無いのです。

そのため、DataGridではセル部分の横結合関連処理はすべて自前で実装しています。

ちなみに、セルの縦結合も対応しています（propsが`data: readonly Data[];`ではだめになってくるので、分かりやすさのためにHierarchicalDataGridという別のコンポーネントに縦結合機能を分離しています）。

## ではTanStack Tableの何の機能を使っているのか

ここまで、TanStack Tableが提供してくれる機能はそこまで要らないし、逆に我々が実装したい機能はそこまでTanStack Tableのサポートが厚くなかったことを紹介しました。

では、何のためにTanStack Tableを使っているのでしょうか。

現在、DataGridでTanStack Tableの機能を活用している点は1つに集約されます。それは**列のリサイズ機能**です。

![列のリサイズ機能を説明する画像。列と列の境目の上にマウスカーソルを乗せたことで横移動可能を示す形にカーソルが変化していることを示している。](/images/kaonavi-datagrid-tanstack-table/resizing.png)

列のリサイズ機能では、ユーザーが列と列の境目を動かして列を変更することができます。TanStack Tableが内部的に列の大きさをステートに持っていてそれを反映する機能です。また、ユーザーのインタラクションに対する対応を全部TanStack Tableがやってくれます。

これを自前で実装するのが大変そうなので、ほとんどこれだけのためにTanStack Tableを使っています。

逆に言えば、ここの実装を頑張って用意するのであれば、TanStack Tableを使わなくても自前で全部いけると思います。

ちなみに、DataGridにはバーチャルスクロール機能も用意していますが、それは **[TanStack Virtual](https://tanstack.com/virtual/latest)** を使っています。同じTanStackシリーズですが、VirtualとTableがめちゃくちゃ相性がいいというわけではなく、普通です。TanStack Virtualだけ使うのでもOKでしょう。

## まとめ

この記事では、DataGridというコンポーネントを実装するにあたって内部実装にTanStack Tableを用いて感じた所感を説明しました。

腐敗防止層を意識しつつ、アクセサ列を中心としたTanStack Tableの機能にそこまで必要性を感じず使わなかった結果、TanStack Tableの機能をほとんど活かさない実装となりました。唯一列のリサイズ機能が便利なので使っています。

DataGridのような複雑なコンポーネントを実装する際はこのようなヘッドレスUIライブラリに頼りたくなりますが、意外と無くてもいけることが分かりました。
