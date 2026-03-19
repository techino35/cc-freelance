---
title: "GASでスプレッドシートの編集をGoogleチャット/Slackに自動通知する方法"
emoji: "🔧"
type: "tech"
topics: ["GAS", "GoogleAppsScript", "自動化", "スプレッドシート"]
published: true
---

## はじめに

チームで共有スプレッドシートを使っていると、「誰かが更新したかどうか」をいちいち確認するのが面倒です。更新のたびに手動でSlackに投稿するのは手間がかかり、抜け漏れも発生します。Google Apps Script（GAS）を使えば、スプレッドシートの編集を検知して自動でSlackやGoogle Chatに通知を飛ばすことができます。

---

## 完成イメージ

スプレッドシートのC列（作業時間）が編集されると、以下のようなメッセージがSlackに自動送信されます。

```
田中 さんが 03/19 の作業時間を更新しました: 8時間
```

通知を受け取ったチームメンバーは、スプレッドシートを開くことなく状況を把握できます。

---

## 実装手順

### ステップ1: スプレッドシートの準備

以下の列構成でスプレッドシートを用意します。

| A列 | B列 | C列 |
|-----|-----|-----|
| 名前 | 日付 | 作業時間 |
| 田中 | 2026/03/19 | 8 |

2行目以降がデータ行です。1行目はヘッダーとして使用します。

### ステップ2: GASエディタを開く

1. スプレッドシートのメニューから「拡張機能」→「Apps Script」を選択
2. デフォルトのコード（`myFunction`）を削除して、以下のコードを貼り付ける

### ステップ3: onEditトリガーのコードを追加する

```javascript
function onEdit(e) {
  var sheet = e.range.getSheet();
  var row = e.range.getRow();
  var col = e.range.getColumn();
  if (col !== 3 || row < 2) return; // C列2行目以降を監視

  var name = sheet.getRange(row, 1).getValue();
  var date = sheet.getRange(row, 2).getValue();
  var hours = e.range.getValue();

  var message = name + " さんが " + Utilities.formatDate(date, "Asia/Tokyo", "MM/dd") + " の作業時間を更新しました: " + hours + "時間";

  // Slack通知
  var webhookUrl = PropertiesService.getScriptProperties().getProperty("SLACK_WEBHOOK");
  if (webhookUrl) {
    UrlFetchApp.fetch(webhookUrl, {
      method: "post",
      contentType: "application/json",
      payload: JSON.stringify({ text: message })
    });
  }
}
```

**コードのポイント:**

- `col !== 3 || row < 2` の条件でC列2行目以降だけを監視し、ヘッダー行や関係ない列の編集では通知しません
- `PropertiesService.getScriptProperties()` でWebhook URLをスクリプトプロパティから取得しています。URLをコードに直書きしないことでセキュリティを確保します

### ステップ4: Slack Webhook URLを設定する

**Slack Incoming Webhookの取得手順:**

1. [Slack API](https://api.slack.com/apps) にアクセスし「Create New App」
2. 「From scratch」を選択してアプリ名とワークスペースを設定
3. 左メニュー「Incoming Webhooks」を有効化
4. 「Add New Webhook to Workspace」から通知先チャンネルを選択
5. 生成されたWebhook URL（`https://hooks.slack.com/services/...`）をコピー

**GASスクリプトプロパティへの登録手順:**

1. GASエディタの左メニュー「プロジェクトの設定」（歯車アイコン）を開く
2. 「スクリプト プロパティ」セクションで「プロパティを追加」
3. プロパティ名: `SLACK_WEBHOOK`、値: コピーしたWebhook URLを入力
4. 「スクリプト プロパティを保存」

**Google Chat Webhookを使う場合:**

Google Chatの場合も同様の手順です。

1. 通知先のスペースを開き、スペース名横の「∨」→「アプリと統合」を選択
2. 「Webhookを追加」からWebhook URLを取得
3. スクリプトプロパティのキー名を `GCHAT_WEBHOOK` として保存

Google Chat用にコードを変更する場合、`payload` の形式が異なります。

```javascript
// Google Chat用のpayload形式
payload: JSON.stringify({ text: message })
// ※ Google ChatもSlackと同じJSON形式で動作します
```

### ステップ5: インストーラブルトリガーを設定する

`onEdit` は単純トリガーとして動作しますが、`UrlFetchApp`（外部HTTP通信）を使う場合はインストーラブルトリガーが必要です。

1. GASエディタ左メニューの「トリガー」（時計アイコン）を開く
2. 右下の「トリガーを追加」をクリック
3. 以下の設定を行う:
   - 実行する関数: `onEdit`
   - イベントのソース: 「スプレッドシートから」
   - イベントの種類: 「編集時」
4. 「保存」をクリック（Googleアカウントの認証画面が出たら許可する）

これで、C列の編集時にSlackへの自動通知が動作します。

---

## 応用: メール通知への拡張

Slackを使っていない場合は、メール通知に変更できます。`onEdit` 関数の末尾に以下を追加します。

```javascript
// メール通知（Slack通知の代替または併用）
function sendEmailNotification(message) {
  var email = PropertiesService.getScriptProperties().getProperty("NOTIFY_EMAIL");
  if (!email) return;

  GmailApp.sendEmail(
    email,
    "[スプレッドシート更新通知] 作業時間が更新されました",
    message,
    {
      htmlBody: "<p>" + message + "</p><p><a href='https://docs.google.com/spreadsheets/d/" + SpreadsheetApp.getActiveSpreadsheet().getId() + "'>スプレッドシートを開く</a></p>"
    }
  );
}
```

`onEdit` 関数内から `sendEmailNotification(message)` を呼び出すだけで、メールでも通知を受け取れます。スクリプトプロパティに `NOTIFY_EMAIL` キーで送信先メールアドレスを登録してください。

---

## まとめ

- `onEdit` トリガーで特定列の編集を検知できる
- `PropertiesService` でWebhook URLを安全に管理できる
- インストーラブルトリガーを使えば外部API連携も可能
- Slack・Google Chat・メールに対応できる

スプレッドシートの更新通知を自動化することで、チームのコミュニケーションロスを大幅に削減できます。

---
## テンプレートを使えば5分で導入できます

この記事の内容をすぐに使いたい方向けに、設定済みのGoogle Sheetsテンプレートを用意しました。
GASコード付き・Instructionsタブ付きで、コピーして設定を変えるだけで動きます。

👉 [Auto-Notify Timesheet Template（Gumroad）](https://inoshu.gumroad.com/l/pknnt)
