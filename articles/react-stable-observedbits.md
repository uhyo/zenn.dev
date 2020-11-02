---
title: "Stable ReactでもobservedBitsを使いたい！"
emoji: "☯️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

**observedBits**とは、ReactのContextに隠された機能です。

まずReactのContextについて復習しておくと、これはコンポーネントツリーの上のほうに配置した**Provider**に与えられた値をそれより下の任意のコンポーネントで取り出すことができる機能です。Reactにおける最も単純なデータの受け渡しはpropsによるものですが、propsは親子間のデータの受け渡ししかできません。すごく上のほうにあるコンポーネントが持っているデータをすごく下のほうにあるコンポーネントで使いたい場合、propsでデータを受け渡すと上から下までの間に経由するコンポーネントの全てでpropsによるデータの受け渡しが必要になってしまいます（いわゆるpropsのバケツリレー）。Contextを使えば、途中のコンポーネントで何もしなくても、上のほうのコンポーネントが持っているデータを下の方のコンポーネントがダイレクトに受け取ることができます。これは実装が簡単になるだけでなく、パフォーマンス的にも有利です（Contextで渡されるデータが変化した場合は途中のコンポーネントの再レンダリングが必要ないため）。

Contextからデータを受け取るコンポーネントは、Contextを通じてやってくるデータが変化した場合は再レンダリングされます。このため、一つのContextを通じてたくさんのデータを送るのは一般に避けるべきです。というのも、たくさんのデータを一つのContextでまとめて送る場合はたくさんのコンポーネントがそのデータをContextから受け取ることになりますが、データが多ければ多いほど各々のコンポーネントにとってはContextの中のデータのうち自分と関係のないデータがの割合が増えることになります。Contextが一つの場合、あるコンポーネントとは関係のないデータのみが変わった場合でも、そのコンポーネントは再レンダリングされてしまいます。これは無駄な再レンダリングであり、パフォーマンスに悪影響を与えます。

データが増えることに対する一般的な対処法は、データを分割して複数のContextに分けて送ることです。そうすることで、コンポーネントは自身と関係のあるデータを含んだContextのみを利用することができ、無駄な再レンダリングを抑制できます。

ところが、Reactの**observedBits**という機能を使うと、Contextが一つのままで無駄な再レンダリングを抑制することができます。ただし、最初に注意しておくと、これは**unstable**な機能です。つまり、実装されてはいるものの公式ドキュメントにも載っていない非公式な機能であり、将来的に消される可能性もあるため無闇に使うべきではないということです。

この記事では、observedBitsはunstableなAPIを使わなくても素直に再現できるよ！　ということを説明します。

## observedBitsの解説

まずobservedBitsについて解説しておきます。次の例をご覧ください。（実際に動作するサンプルへのリンクはあとで記載します。）

```tsx
const DataContext = createContext(
  { foo: 0, bar: 0 },
  (prev, next) =>
    (+(prev.foo !== next.foo) * 1) | (+(prev.bar !== next.bar) * 2)
);

const NumberDisplays: React.FC = () => {
  return (
    <div className="NumberDisplays">
      <DataContext.Consumer unstable_observedBits={1}>
        {(data) => <NumberDisplay label="foo" data={data.foo} />}
      </DataContext.Consumer>
      <DataContext.Consumer unstable_observedBits={2}>
        {(data) => <NumberDisplay label="bar" data={data.bar} />}
      </DataContext.Consumer>
      <DataContext.Consumer unstable_observedBits={3}>
        {(data) => (
          <NumberDisplay label="foo + bar" data={data.foo + data.bar} />
        )}
      </DataContext.Consumer>
    </div>
  );
};
```

まず`DataContext`というContextを`createContext`で作っています。ここで`createContext`の第2引数に渡されている関数がchangedBitsを計算する関数です。changedBitsというのは**データのうちどの部分が変更されたのかを表す数値**です。この関数はデータが変わったときに前のデータと今のデータを引数に呼び出され、changedBitsを返します。この例では`DataContext`に入るのは`{ foo: number; bar: number }`という形のオブジェクトですが、この関数が返すchangedBitsは、`foo`のみが変わったなら`1`、`bar`のみが変わったなら`2`、両方が変わったなら`3`です。

Bitsという名前から推測できるように、changedBitsはビットパターンとして解釈されます。先ほどの関数が返すchangedBitsをよく見ると、1の位（一番下のビット）は`foo`が変わったときに1となり、2の位（下から2番目のビット）は`bar`が変わるときに1となります。

上の例では`DataContext.Provider`（`DataContext`にデータを提供するコンポーネント）は出てきませんが、どこか上の方で使われていると考えてください。例の下半分では`DataContext.Consumer`が使われており（これが`DataContext`からデータを受け取るためのコンポーネントです）、`unstable_observedBits`というpropが渡されています。

通常のConsumerはContextに流れるデータが変わったら常に再レンダリングしますが、`unstable_observedBits`が指定されているConsumerはデータの変化時、**changedBitsとobservedBitsの両方で1となるビットがある場合**（言い換えれば`changedBits & observedBits`が`0`でない場合）にのみ再レンダリングされます。別の言い方をすれば、このobservedBitsはchangedBitsに対するビットマスクとして働き、マスク後のchangedBitsが0となった場合はデータの変化を無視して再レンダリングを行いません。

上の例では、最初の`DataContext.Consumer`は`data.foo`のみを使用するため、`foo`が変化したときにのみ再レンダリングを行うようにobservedBitsを`1`としています。`bar`のみが変化したときはchangedBitsが2、observedBitsが1となるため、`2 & 1`が0となり再レンダリングが発生しません。

次のConsumerは`data.bar`のみを使用するためobservedBitsを2としています。最後のConsumerは`data.foo`と`data.bar`の両方を使っているため、どちらの変化にも反応できるようにobservedBitsを3としています。

## unstableなAPIを使わずにobservedBitsを再現する

さて、ここからが本題です。observedBitsは便利ですが、unstableなAPIなので使いたくありません。そこで今回、observedBitsを通常のContextを用いて再実装していました。ポイントは、**observedBitsの機能は普通のContextを32個使えば再現できる**ということです。

というのも、observedBitsの本質は**一つのContextの中に複数のチャンネルが混在している**という点にあります。changedBitsやobservedBitsの各ビットはそれぞれ異なるチャンネルを表しており、changedBitsはどのチャンネルに対して更新をかけるか制御していると捉えられます。一方、observedBitsはどのチャンネルから更新を受け取るかを制御しています。

そして、そのチャンネルは32個あります。なぜなら、JavaScriptのビット演算は32ビットまでサポートしているからです。

ここでチャンネルと呼んでいるものは、上からデータを流すことができて下ではそれを受け取ることができる（コンポーネントが再レンダリングされる）というものです。これはよく考えると、通常の（observedBitsを使わない）Contextの機能そのものですね。ということで、**observedBitsの使用は普通のContextを32個使用するのと本質的に同じことです。**

ということで、32個のContextを使ってobservedBitsを再現してみました。できたものがこちらです。

https://codesandbox.io/s/stable-observedbits-dyx92?file=/src/Ours.tsx

@[codesandbox](https://codesandbox.io/embed/stable-observedbits-dyx92?fontsize=14&hidenavigation=1&theme=dark)

このアプリの中ではReactから提供されているunstable_observedBitsと今回実装したものの両方で同じものが実装されており、どちらも同じように動作することが確かめられます。今回実装したものは`createBitsContext`でコンテキストを作るようになっています。

実装の中身を覗いてみましょう。もちろんハイライトは[ここ](https://codesandbox.io/s/stable-observedbits-dyx92?file=/src/BitsContext/BitsContext.tsx:2462-2475)です。

```tsx
return (
  <Context1.Provider value={data[0]}>
    <Context2.Provider value={data[1]}>
      <Context3.Provider value={data[2]}>
        <Context4.Provider value={data[3]}>
          <Context5.Provider value={data[4]}>
            <Context6.Provider value={data[5]}>
              <Context7.Provider value={data[6]}>
                <Context8.Provider value={data[7]}>
                  <Context9.Provider value={data[8]}>
                    <Context10.Provider value={data[9]}>
                      <Context11.Provider value={data[10]}>
                        <Context12.Provider value={data[11]}>
                          <Context13.Provider value={data[12]}>
                            <Context14.Provider value={data[13]}>
                              <Context15.Provider value={data[14]}>
                                <Context16.Provider value={data[15]}>
                                  <Context17.Provider value={data[16]}>
                                    <Context18.Provider value={data[17]}>
                                      <Context19.Provider value={data[18]}>
                                        <Context20.Provider
                                          value={data[19]}
                                        >
                                          <Context21.Provider
                                            value={data[20]}
                                          >
                                            <Context22.Provider
                                              value={data[21]}
                                            >
                                              <Context23.Provider
                                                value={data[22]}
                                              >
                                                <Context24.Provider
                                                  value={data[23]}
                                                >
                                                  <Context25.Provider
                                                    value={data[24]}
                                                  >
                                                    <Context26.Provider
                                                      value={data[25]}
                                                    >
                                                      <Context27.Provider
                                                        value={data[26]}
                                                      >
                                                        <Context28.Provider
                                                          value={data[27]}
                                                        >
                                                          <Context29.Provider
                                                            value={data[28]}
                                                          >
                                                            <Context30.Provider
                                                              value={
                                                                data[29]
                                                              }
                                                            >
                                                              <Context31.Provider
                                                                value={
                                                                  data[30]
                                                                }
                                                              >
                                                                <Context32.Provider
                                                                  value={
                                                                    data[31]
                                                                  }
                                                                >
                                                                  {children}
                                                                </Context32.Provider>
                                                              </Context31.Provider>
                                                            </Context30.Provider>
                                                          </Context29.Provider>
                                                        </Context28.Provider>
                                                      </Context27.Provider>
                                                    </Context26.Provider>
                                                  </Context25.Provider>
                                                </Context24.Provider>
                                              </Context23.Provider>
                                            </Context22.Provider>
                                          </Context21.Provider>
                                        </Context20.Provider>
                                      </Context19.Provider>
                                    </Context18.Provider>
                                  </Context17.Provider>
                                </Context16.Provider>
                              </Context15.Provider>
                            </Context14.Provider>
                          </Context13.Provider>
                        </Context12.Provider>
                      </Context11.Provider>
                    </Context10.Provider>
                  </Context9.Provider>
                </Context8.Provider>
              </Context7.Provider>
            </Context6.Provider>
          </Context5.Provider>
        </Context4.Provider>
      </Context3.Provider>
    </Context2.Provider>
  </Context1.Provider>
);
```

確かに32個のProviderが使われていますね[^loop]。

[^loop]: ループでこれを書けることは知っています。インパクト重視の記事なので、ループ使えよというつっこみはご容赦ください。

大まかな実装としては、大元のProviderに提供されたデータが変わったときにchangedBitsを計算し、それに応じて32個のProviderのうちどれのデータを更新するか決める感じになっています。Consumerの側も、32個のContextのうちobservedBitsが立っているもののみをsubscribeする実装になっています。やや実装がごちゃごちゃしていますが、本質的にやっていることはこれだけです。

## おまけ: React 18（仮）でもobservedBitsを使いたい！

ところで、ここまでuseContextの話が出てきませんでしたね。useContextというのはContextからデータを受け取るもう一つの方法であり、フックなので`Context.Consumer`よりも今どきです。useContextでもobservedBitsは一応サポートされているのですが（useContextの第2引数にobservedBitsを指定できます）、使用するとdevelopmentビルドではコンソールに警告が表示されます（Consumerの`unstable_observedBits`はunstableとはいえ警告が表示されることはありません）。これは`unstable_observedBits`よりもさらに不安定で、使うのは非常に危ないことを意味しています。実質`useContext`はobservedBitsに対応していないと言っても良いですね。いやあ良かった良かった。なぜ良かったのかといえば、今回の実装は`useContext`には対応していないし対応がとても難しいからです[^note_useContext]。

[^note_useContext]: 厳密には、`obsevedBits`が動的に変化しないという制約を付ければ簡単です（32個のうち対応するものに対してuseContextを並べればよいので）。`observedBits`が変化する場合に対応することができません。

とはいえ、これからはフックの時代ですからフックもでobservedBitsを使いたいという人もいるかもしれません。残念ながら、現行のReactではステート管理ライブラリを1つ作るような大掛かりな実装が必要になってしまい面白みがありません。そこで、今回はまだ現行のReactには導入されていない新しいフックである[useMutableSource](https://github.com/reactjs/rfcs/blob/master/text/0147-use-mutable-source.md)を使います。これはReactのexperimentalビルドを使用すると試してみることができます。

`useMutableSource`はコンポーネントツリーの外部のサブスクリプションを簡単に実装できるフックです。React 18でリリースされるような気がしますが確証はありません。ともあれ、外部ライブラリを使わなくてもサブスクリプションができるのはとても嬉しいですね[^useMutableSource_ConcurrentMode]）。Providerを32個使うというアイデアからは離れますが、サブスクリプションベースにすることで比較的簡単に実装することができます。やっていることは、データの変更をsubscribeするときにobservedBitsも一緒に登録することで、それに合致する更新があったときだけコールバック関数が呼び出されるという単純なものです。この実装だとコード中に`changedBits & observedBits`が現れてとても美しいです。

[^useMutableSource_ConcurrentMode]: もっとも、それ自体は既存のライブラリを使えばできることなので、どちらかといえば`useMutableSource`の本領はConcurrent Mode下でもうまく動くことにあります。

ということで、実際にやってみたのがこちらです。

https://codesandbox.io/s/stable-observedbits-usemutablesource-ey0m1?file=/src/BitsContext/BitsContext.tsx

@[codesandbox](https://codesandbox.io/embed/stable-observedbits-usemutablesource-ey0m1?fontsize=14&hidenavigation=1&theme=dark)

`useMutableSource`の使い方を詳しく解説したりはしませんが、未来のReactではこんなこともできるよという例でした。observedBitsの実装も、最初のものと比べると`useMutableSource`を使うほうが行数が半分以下になっていますね。

## まとめ

この記事では、Reactのunstableな機能であるobservedBitsは実は普通のContextを32個使えば再現できるよということを紹介しました。記事の冒頭で、Contextのデータが大きくなったら複数のContextに分けるのがいいと述べましたが、observedBitsを使ったとしてもやっていることは本質的には同じなのです。もちろん、ステートがとても複雑な場合はContextで済ませるのではなくステート管理ライブラリの使用も検討するとよいでしょう。