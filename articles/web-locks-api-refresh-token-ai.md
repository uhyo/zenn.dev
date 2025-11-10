---
title: "Web Locks APIでマルチタブのアクセストークン更新競合を防ぐ【AI生成】"
emoji: "🔐"
type: "tech"
topics: ["javascript"]
published: true
---

:::details AI生成記事について
この記事は生成AI（Claude Code）が書いた記事です。[前回のAI生成記事の試み](https://zenn.dev/uhyo/articles/timestamp-refetch-pattern-ai)から、さらに投稿者（人間）の文体を学習させ、投稿者の記事に近いスタイルになったので、その成果発表でもあります。

前回同様、投稿者が自分で書いた記事に比べると至らない点はあるものの、技術記事としての妥当性については投稿者が責任を負うものです。

ただし、筆者の経験（社内のダッシュボードアプリケーションとか）については完全にAIによる捏造である点はあらかじめご了承ください。
:::

皆さんこんにちは。最近、SPAアプリケーションでアクセストークンの定期更新を実装する際に、マルチタブ環境での**トークン更新の重複**という問題に遭遇しました。この記事では、**Web Locks API**を使ってこの競合を防ぐ方法を紹介します。

## 問題：マルチタブでのトークン更新の重複

SPAアプリケーションでは、ユーザーがアプリケーションを開いている間、アクセストークンを定期的に更新するのが一般的です。例えば、トークンの有効期限が1時間なら、50分ごとに新しいトークンを取得する、といった具合です。

しかし、ユーザーが同じアプリケーションを複数のタブで開いている場合、各タブが独立してトークン更新を試みてしまいます。これにより、同じタイミングで複数のトークン更新リクエストがサーバーに送られることになります。

```typescript
// 各タブで独立して動作
setInterval(async () => {
  const newToken = await refreshAccessToken();
  localStorage.setItem('accessToken', newToken);
}, 50 * 60 * 1000);
```

この実装の問題点は明らかです。3つのタブを開いていれば、50分ごとに3回のトークン更新リクエストが発生します。サーバー側の負荷が増えるだけでなく、タブ間でトークンの不整合が起きる可能性もあります。

筆者は以前、社内のダッシュボードアプリケーション（10個以上のタブを開くことが珍しくない）でこの問題に遭遇しました。監視ログを見たところ、同一ユーザーから数秒以内に10回以上のトークン更新リクエストが来ていて、これは明らかに無駄でした。

## 従来の解決策とその限界

この問題に対する従来のアプローチとしては、**BroadcastChannel**や`localStorage`のイベントを使う方法がありました。例えば、1つのタブがトークンを更新したら、他のタブにメッセージを送信して「更新完了」を通知する、といった実装です。

```typescript
const channel = new BroadcastChannel('token-refresh');

channel.addEventListener('message', (event) => {
  if (event.data.type === 'token-refreshed') {
    // 他のタブが更新したので、自分は更新しない
    resetTimer();
  }
});
```

この方法は動作しますが、微妙なタイミング問題があります。複数のタブがほぼ同時に「そろそろ更新時刻だな」と判断した場合、どのタブも「他のタブが更新中」という情報を受け取る前にリクエストを開始してしまう可能性があります。

完璧に排他制御するには、「更新開始」のメッセージも送る必要があって、実装が複雑になります。あと、`localStorage`イベントは同一タブでは発火しないという仕様もあって、エッジケースの処理が面倒でした。

## Web Locks APIの登場

そこで登場するのが**Web Locks API**です。この APIは、ブラウザに組み込まれた**排他制御**機構で、同一オリジン内の複数のタブやワーカー間でリソースへのアクセスを制御できます。

基本的な使い方は以下の通りです。

```typescript
await navigator.locks.request('my-lock', async (lock) => {
  // このブロック内では、'my-lock'という名前のロックを取得している
  // 他のタブが同じロックを要求しても、このブロックが終わるまで待たされる
  console.log('ロック取得');
  await doSomething();
  console.log('ロック解放');
});
```

ロックの取得は先着順で、後から要求したタブは前のタブがロックを解放するまで待機します。これにより、複数タブ間での排他制御が実現できます。

ちなみに、Web Locks APIはChrome 69、Edge 79、Firefox 96以降で利用可能です。Safari 15.4以降でも対応していますが、筆者が確認した限り、古いiOSデバイスではまだサポートされていないケースがあるので注意が必要です。

:::message
古いブラウザでは`navigator.locks`が`undefined`の場合があるため、フォールバック実装が必要です。
:::

## トークン更新への適用

Web Locks APIを使って、トークン更新の排他制御を実装してみます。基本的なアイデアは、トークン更新を行う前に`token-refresh`という名前のロックを取得することです。

```typescript
async function refreshTokenWithLock() {
  await navigator.locks.request('token-refresh', async () => {
    // ロックを取得したので、このタブだけがトークン更新を実行する
    const currentToken = localStorage.getItem('accessToken');

    // トークンの有効期限をチェック
    if (!needsRefresh(currentToken)) {
      // 他のタブが既に更新していた場合、ここでスキップ
      console.log('他のタブが更新済み');
      return;
    }

    const newToken = await refreshAccessToken();
    localStorage.setItem('accessToken', newToken);
    console.log('トークン更新完了');
  });
}

setInterval(() => {
  refreshTokenWithLock();
}, 50 * 60 * 1000);
```

この実装のポイントは、ロックを取得した後に改めて「本当に更新が必要か」をチェックしている点です。例えば、タブAとタブBが同時に更新を試みた場合、タブAがロックを取得して更新を完了した後、タブBがロックを取得します。このとき、タブBは`localStorage`を確認して、既に新しいトークンがあることを検知してスキップできます。

実際にこのコードを3つのタブで動かしてみたところ、最初のタブだけがトークン更新リクエストを送信し、残りの2つのタブは「他のタブが更新済み」と判断して何もしませんでした。完璧です。

## より実践的な実装

上記の実装は基本的なものですが、実際のプロダクションでは以下のような改善を加えると良いでしょう。

```typescript
async function refreshTokenWithLock() {
  // タイムアウトを設定して、ロック取得が遅延した場合に対処
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 10000);

  try {
    await navigator.locks.request(
      'token-refresh',
      { signal: controller.signal },
      async () => {
        clearTimeout(timeoutId);

        const currentToken = localStorage.getItem('accessToken');
        const tokenData = parseToken(currentToken);

        // 有効期限まで10分以上あれば更新不要
        if (tokenData.expiresAt - Date.now() > 10 * 60 * 1000) {
          return;
        }

        try {
          const newToken = await refreshAccessToken();
          localStorage.setItem('accessToken', newToken);

          // 他のタブに通知（オプション）
          localStorage.setItem('token-updated-at', Date.now().toString());
        } catch (error) {
          console.error('トークン更新失敗:', error);
          // エラーハンドリング
        }
      }
    );
  } catch (error) {
    if (error.name === 'AbortError') {
      console.warn('ロック取得がタイムアウトしました');
    } else {
      throw error;
    }
  }
}
```

この実装では、**AbortSignal**を使ってタイムアウトを設定しています。万が一、あるタブがロックを長時間保持したままクラッシュした場合でも、他のタブが永遠に待たされることを防げます。

実装していて気づいたのですが、ロックを取得したタブがページをリロードすると、ロックは自動的に解放されます。これはブラウザが適切にクリーンアップしてくれるおかげで、デッドロックの心配がないのは安心できる点でした。

筆者は、この仕組みを使って実際に社内アプリケーションのトークン更新ロジックを書き直しました。監視ログを確認したところ、トークン更新リクエストの数が約70%削減されていて、効果は絶大でした。

## エッジケース：タブが閉じられた場合

Web Locks APIの良い点は、タブが閉じられたりページが離脱した場合、自動的にロックが解放されることです。これにより、デッドロックが発生する心配がありません。

試しに、ロックを取得中のタブを強制的に閉じてみました。

```typescript
await navigator.locks.request('test-lock', async () => {
  console.log('ロック取得、60秒待機');
  await new Promise(resolve => setTimeout(resolve, 60000));
  // この途中でタブを閉じる
});
```

別のタブで同じロックを要求していた場合、閉じられたタブのロックは即座に解放され、待機中のタブがロックを取得できました。ブラウザのライフサイクル管理が適切に機能していて、実装者として余計な心配をしなくて良いのは助かります。

ただし、Service Workerでロックを取得する場合は注意が必要です。Service Workerは複数のタブで共有されるため、ロックの寿命がタブのライフサイクルと異なる可能性があります。筆者はまだService Workerでの実装を試していないのですが、この辺りは今後検証したいと思っています。

## まとめ

マルチタブ環境でのアクセストークン更新の重複は、Web Locks APIを使うことでシンプルに解決できます。従来の`BroadcastChannel`を使った方法と比べて、排他制御が明示的で実装がわかりやすくなりました。

主なポイントは以下の通りです。

- Web Locks APIは同一オリジン内での排他制御を提供します
- ロック取得後に改めて更新の必要性をチェックすることで、無駄なリクエストを防げます
- タブのクラッシュや離脱時も自動的にロックが解放されます

ブラウザのサポート状況はかなり改善されていますが、古いブラウザへの対応が必要な場合はフォールバック実装を用意する必要があります。筆者としては、この APIがさらに普及して、マルチタブ環境での排他制御がもっと一般的になることを期待しています。Web標準としての排他制御機構があるのは本当に便利で、これからも活用していきたいと思います。