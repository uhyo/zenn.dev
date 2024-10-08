---
title: 裏の世界・ステート更新・副作用
---

次は、世界が分岐している状態でのステート更新の挙動にもう少し深入りしてみます。

この章の内容を反映済みのコードは`chapter/state-updates`ブランチにあります。

今回トランジションとして行なっているステート更新は`setCounter((c) => c + 1)`でした。このように関数を用いてステートを更新する場合、この関数に副作用を含んではいけないということはReact使いならば直感的に理解できると思います。この章では、その理由の一端が明らかになります。

# ステート更新が発生する瞬間を調べる

```diff tsx
 function App() {
   const [counter, setCounter] = useState(0);
   const time = useTime();
   return (
     <div className="text-center">
       <h1 className="text-2xl">React App!</h1>
       <p className="tabular-nums">🕒 {time}</p>
       <Suspense fallback={<p>Loading...</p>}>
         <ShowData dataKey={counter} />
       </Suspense>
       <p>
         <button
           className="border p-1"
           onClick={() => {
             startTransition(() => {
+              setCounter((c) => {
+                console.log(c, "→", c + 1);
+                return c + 1;
+              });
             });
           }}
         >
           Counter is {counter}
         </button>
       </p>
     </div>
   );
 }
```

ということで、`setCounter`のステート更新関数に`console.log`を仕込んでみましょう。

このとき、皆さんが期待するのはどのような挙動でしょうか。普通に考えると、「ボタンを押した瞬間に1回表示される」ではないかと思います。しかし、実際に試してみるとそうはならないはずです。

---

実際にボタンを押してみると、0.1秒ごとに`console.log`が実行されて全部で11回ほどログが表示されるのがお分かりになるはずです。なぜこんな挙動になるのでしょうか。

これを正確に理解するには、前章の最後に出てきた図を少し修正しなければいけません。より正確な図は次のようになります。裏の世界でどのようにステートが遷移しているのかは変わりませんが、その計算の過程が異なります。

![ステート計算の実際の挙動を考慮した模式図](/images/react-concurrent-handson-2/state-updates-1.png)

このように、裏の世界のステートというのは、表の世界のステートが更新された際には毎回新たに計算し直されています。今回、表の世界と裏の世界の差分というのはトランジション`setCounter((c) => c + 1)`が適用されているかどうかでした。ですから、Reactはこの差分を表の世界と裏の世界の違いとして覚えておいて、表の世界が更新された際はこの差分を再適用する事で裏の世界のステートを得るのです。そのため、`console.log`を仕込んだ先ほどの例では、表の世界のステートが変わるたびに関数が呼び出されているのが観測できたのです。

ですから、前章での説明は直感的な理解しやすさを優先してやや正確さに欠けていました。実際には、分岐した裏の世界で独自にステート管理が行われているというよりは、実際に保存されているのは差分であり裏の世界は表の世界に差分を適用して作られる世界だったのです。

トランジションによる世界の分岐はよくgitのブランチに例えられますが、表の世界から裏の世界を作るステートの差分がコミットに相当し、表の世界ブランチが更新されたときは裏の世界ブランチは新しい表の世界ブランチにrebaseされると理解できます。

ちなみに、トランジションとしてuseReducerを用いたステート更新をした場合も、そのreducer関数の呼び出しが差分として保存され、表の世界が更新されるたびにreducer関数が再呼び出しされます。このことから、reducer関数にもやはり副作用を含んではいけないことが分かります。もしuseStateやuseReducerのステート更新に副作用があった場合、それはトランジションを発生させた瞬間の1回だけ起こるのではなく、トランジションが完了するまでの間に複数回呼び出されるかもしれないからです。

# ステート更新と関数の純粋性

以上の説明から、ステート更新に使用する関数についてはその**純粋性**が重要であることが分かります。副作用を起こさないこともその一部ですが、他にも同じ引数なのに返り値が毎回変わってはいけないということも重要です。例えば、`setCounter`で増えるカウンターの値がランダムであるとしてみましょう。

```diff tsx
 function App() {
   const [counter, setCounter] = useState(0);
   const time = useTime();
   return (
     <div className="text-center">
       <h1 className="text-2xl">React App!</h1>
       <p className="tabular-nums">🕒 {time}</p>
       <Suspense fallback={<p>Loading...</p>}>
         <ShowData dataKey={counter} />
       </Suspense>
       <p>
         <button
           className="border p-1"
           onClick={() => {
             startTransition(() => {
               setCounter((c) => {
+                const next = c + 1 + Math.floor(Math.random() * 10);
+                console.log(c, "→", next);
+                return next;
               });
             });
           }}
         >
           Counter is {counter}
         </button>
       </p>
     </div>
   );
 }
```

こうすると、ボタンを押してからやはり0.1秒間隔で`console.log`が実行されます。もちろん、その度に違う値が（偶然同じ値が続くこともありますが）裏の世界のcounterにセットされます。

しかも、よく見ると今度はちょうど11回（1秒後）で終了するとは限りません。終了（サスペンドが終了して表の世界に新しいcounterの値が反映される）までに発生する再レンダリングは15回くらいかかる場合もあるはずです。

さらにその上、ボタンを続けて押してみると、今度は1秒もかからずにサスペンドが終了するという現象が見られるはずです。一瞬で終わる場合もあれば、0.3秒とかそれくらいかかる場合もあります。

以上のような現象がどうして起きるのか説明できれば、かなりトランジションを理解していると言えるでしょう（そもそもステート更新関数をランダムにしなければいい話ではありますが）。

---

では、例によって図で説明していきます。

![模式図](/images/react-concurrent-handson-2/state-updates-2.png)

今回は、`setCounter`によって裏の世界が作られてサスペンドするところは同じですが、0.1秒ごとに再計算される裏の世界において毎回counterの値が異なってきます（偶然重複することももちろんありますが）。図では最初の3回がcounter=4, counter=1, counter=6となった場合を示しています。

図では、最初のサスペンド（counter=4）が1秒後に完了したものの、その結果（再レンダリング結果）が表の世界に反映されないことを示しています。なぜなら、1秒後の再レンダリングの際も裏の世界のステートが再計算されますが、90%の確率でその際にcounter=4にはならず、レンダリング開始時（サスペンド時）からステートが変わっているからこのレンダリング結果は古いものだと判断されるからです。もしこのタイミングで10%の幸運を引き当てて再計算時もcounter=4だった場合、それが表の世界に反映されて無事レンダリング完了となります。そうでなかった場合、それ以降のサスペンド（counter=1, counter=6）も、それが完了した際の再計算時の裏の世界のcounterが、サスペンド時のcounterと一致していなければ表の世界に反映できません。これが、表の世界に結果が反映されるのにちょうど1秒ではなくそれ以上かかる（ことが多い）理由です。

では、10%の当たりくじを引くまでずっとサスペンドし続けるのでしょうか。実はそうでもありません。もし10%がずっと続くとしたら表の世界に反映されるまでにかかる時間の期待値は1.9秒ですが、実際にはもう少し短く感じられるはずです。

その理由は、各サスペンドにおいて`useData`内で取得したデータはキャッシュされるからです。特に、あるcounterの値でサスペンドしてデータを取得してから1秒経過すれば、そのcounterの値のデータは取得済みとなります。よって、裏の世界のステートを再計算する際にすでに取得済みのデータを持つcounterの値が偶然選ばれた場合、そのレンダリングはサスペンドせずに完了します。よって、その時点で表の世界に反映されます。時間が経つほどそのようなcounterの値が増えていきますので、平均して1.9秒よりは短い時間でそのような値に行き当たってトランジションを完了できるでしょう。

なお、一度レンダリング完了後再度ボタンを押すと一瞬またはかなり短い時間でレンダリングが完了（表の世界に反映）するのも同じ理由で説明できます。つまり、前回色々なcounterの値でサスペンドしたことですでにデータ取得が済んでおり、再びトランジションを起こしたときに選ばれたcounterの値がすでにデータを持っていてサスペンドしないからです。サスペンドしたとしても、1秒もかからずにすでにデータを持ったcounterに行き当たってその時点でトランジションが完了するでしょう。

改めて注意しておくと、この例のようにステート更新関数を純粋ではなくすることは、（勉強のためには良かったですが）このように妙な挙動を引き起こす原因となるためすべきではありません。次の章に進む前に忘れずに元に戻しておきましょう。

```diff tsx
 function App() {
   const [counter, setCounter] = useState(0);
   const time = useTime();
   return (
     <div className="text-center">
       <h1 className="text-2xl">React App!</h1>
       <p className="tabular-nums">🕒 {time}</p>
       <Suspense fallback={<p>Loading...</p>}>
         <ShowData dataKey={counter} />
       </Suspense>
       <p>
         <button
           className="border p-1"
           onClick={() => {
             startTransition(() => {
+              setCounter((c) => c + 1);
             });
           }}
         >
           Counter is {counter}
         </button>
       </p>
     </div>
   );
 }
```

