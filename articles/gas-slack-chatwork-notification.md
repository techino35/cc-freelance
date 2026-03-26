---
title: "GASでSlack・Chatworkに自動通知を送る最小構成【コピペOK】"
emoji: "🔔"
type: "tech"
topics: ["GAS", "Slack", "Chatwork", "自動化", "通知"]
published: false
---

## こんな人向け

- 毎朝スプレッドシートの数字をSlackにコピペしている
- 締め切りリストをChatworkに手動で貼っている
- 「誰かスクリプト書けないの？」と思いながら自分でやり続けている

GASとWebhook/APIを使えば、これらは**コードを書かずにコピペだけで自動化できます**。

この記事ではSlackとChatwork、両方の通知コードを最小構成で解説します。

---

## Slack 自動通知

### Incoming Webhook の設定手順

1. [Slack API](https://api.slack.com/apps) にアクセスし「Create New App」
2. 「From scratch」を選択、App名とワークスペースを設定
3. 左メニューの「Incoming Webhooks」をクリック
4. 「Activate Incoming Webhooks」をオンに切り替える
5. 「Add New Webhook to Workspace」をクリック
6. 通知を送りたいチャンネルを選んで「許可する」
7. 発行された Webhook URL（`https://hooks.slack.com/services/xxxx/xxxx/xxxx`）をコピー

この URL が通知の送り先になります。GAS のスクリプトプロパティに保存しておきましょう。

GASエディタの「プロジェクトの設定」→「スクリプトプロパティ」→「プロパティを追加」から以下を登録します。

| プロパティ名 | 値 |
|---|---|
| SLACK_WEBHOOK_URL | https://hooks.slack.com/services/xxxx |

---

### GASコード（Slack版）

```javascript
// Slack に通知を送る関数
function sendSlackNotification(message) {
  const webhookUrl = PropertiesService.getScriptProperties()
    .getProperty('SLACK_WEBHOOK_URL');

  if (!webhookUrl) {
    Logger.log('エラー: SLACK_WEBHOOK_URL が設定されていません');
    return false;
  }

  const payload = {
    text: message
  };

  const options = {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };

  try {
    const response = UrlFetchApp.fetch(webhookUrl, options);
    const statusCode = response.getResponseCode();

    if (statusCode === 200) {
      Logger.log('Slack通知 送信成功');
      return true;
    } else {
      Logger.log('Slack通知 失敗: ステータスコード ' + statusCode);
      return false;
    }
  } catch (e) {
    Logger.log('Slack通知 例外: ' + e.message);
    return false;
  }
}

// 実行テスト用（GASエディタから直接実行して確認）
function testSlackNotification() {
  sendSlackNotification('GASからのテスト通知です :wave:');
}
```

`testSlackNotification` を実行して、Slackに通知が届けば設定完了です。

#### リッチな通知（アタッチメント付き）

単純なテキストではなく、色付きのカード形式で送りたい場合は以下を使います。

```javascript
function sendSlackRichNotification(title, body, color) {
  const webhookUrl = PropertiesService.getScriptProperties()
    .getProperty('SLACK_WEBHOOK_URL');

  // color: 'good'（緑）, 'warning'（黄）, 'danger'（赤）, '#36a64f'（カラーコード）
  const payload = {
    attachments: [
      {
        color: color || 'good',
        title: title,
        text: body,
        ts: Math.floor(Date.now() / 1000)
      }
    ]
  };

  const options = {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };

  UrlFetchApp.fetch(webhookUrl, options);
}
```

---

## Chatwork 自動通知

### Chatwork API の設定手順

1. [Chatwork](https://www.chatwork.com/) にログイン
2. 右上のアイコン→「サービス連携」→「API Token」
3. 「発行する」をクリックしてAPIトークンをコピー
4. 通知を送りたいルームのURLを確認（`https://www.chatwork.com/#!rid123456` の `123456` がルームID）

GAS のスクリプトプロパティに以下を登録します。

| プロパティ名 | 値 |
|---|---|
| CHATWORK_API_TOKEN | 発行したAPIトークン |
| CHATWORK_ROOM_ID | 通知先ルームのID（数字のみ） |

---

### GASコード（Chatwork版）

```javascript
// Chatwork に通知を送る関数
function sendChatworkNotification(message) {
  const props = PropertiesService.getScriptProperties();
  const apiToken = props.getProperty('CHATWORK_API_TOKEN');
  const roomId = props.getProperty('CHATWORK_ROOM_ID');

  if (!apiToken || !roomId) {
    Logger.log('エラー: CHATWORK_API_TOKEN または CHATWORK_ROOM_ID が未設定です');
    return false;
  }

  const url = 'https://api.chatwork.com/v2/rooms/' + roomId + '/messages';

  const options = {
    method: 'post',
    headers: {
      'X-ChatWorkToken': apiToken
    },
    payload: {
      body: message,
      self_unread: 0  // 自分の投稿を未読にしない
    },
    muteHttpExceptions: true
  };

  try {
    const response = UrlFetchApp.fetch(url, options);
    const statusCode = response.getResponseCode();

    if (statusCode === 200) {
      Logger.log('Chatwork通知 送信成功');
      return true;
    } else {
      Logger.log('Chatwork通知 失敗: ステータスコード ' + statusCode);
      Logger.log('レスポンス: ' + response.getContentText());
      return false;
    }
  } catch (e) {
    Logger.log('Chatwork通知 例外: ' + e.message);
    return false;
  }
}

// 実行テスト用
function testChatworkNotification() {
  sendChatworkNotification('GASからのテスト通知です');
}
```

Chatwork では `[To:ユーザーID]` を本文に含めることでメンションを送れます。

```javascript
// メンション付き通知
function sendChatworkMention(userId, message) {
  const fullMessage = '[To:' + userId + '] ' + message;
  sendChatworkNotification(fullMessage);
}
```

---

## 応用例：スプレッドシートの値が変わったら通知

スプレッドシートのB2セル（例：今日の売上）が前日から変化したときだけ通知する実装例です。

```javascript
// スプレッドシートの値を監視して通知
function checkAndNotify() {
  const props = PropertiesService.getScriptProperties();
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName('集計');

  // 今日の値を取得
  const todayValue = sheet.getRange('B2').getValue();
  const label = sheet.getRange('A2').getValue(); // 例：「今日の売上」

  // 前回の値をスクリプトプロパティから取得（初回はnull）
  const lastValue = props.getProperty('LAST_VALUE');

  if (lastValue !== null && String(todayValue) !== lastValue) {
    // 値が変わったら通知
    const message = [
      label + ' が更新されました',
      '前回: ' + lastValue,
      '今回: ' + todayValue,
      '更新日時: ' + Utilities.formatDate(new Date(), 'Asia/Tokyo', 'yyyy/MM/dd HH:mm')
    ].join('\n');

    sendSlackNotification(message);
    // Chatworkに送る場合は下記を使う
    // sendChatworkNotification(message);
  }

  // 今回の値を記録
  props.setProperty('LAST_VALUE', String(todayValue));
}
```

このスクリプトを「時間ベースのトリガー」で毎時1回実行するように設定すれば、値が変わった瞬間に自動通知が届きます。

GASのトリガー設定手順は以下です。

1. GASエディタ左側の時計アイコン（トリガー）をクリック
2. 「トリガーを追加」→ 実行する関数を `checkAndNotify` に設定
3. イベントのソース「時間主導型」→「時間ベースのタイマー」→「1時間おき」

---

## まとめ

- Slack は Incoming Webhook URL をスクリプトプロパティに保存して `UrlFetchApp.fetch` を叩くだけ
- Chatwork は APIトークンとルームIDをヘッダー・URLに埋め込む
- どちらも関数化しておけば他のスクリプトから `sendSlackNotification("テキスト")` の1行で呼び出せる

手動でコピペしている報告作業は、一度GASで自動化してしまえば二度とやらなくて済みます。この記事のコードはそのままコピペで動きます。ぜひ試してみてください。

---

GASを使った業務自動化のコードをまとめたテンプレート集をGumroadで販売しています。Slack/Chatwork通知をベースに、日報自動送信・スプレッドシートバックアップ・フォーム回答通知など10種類以上のすぐに使えるコードが入っています。

https://inoshu.gumroad.com/l/etlfbf
