---
title: "美容サロンのシフト管理をGASで自動化する方法【スタッフ通知付き】"
emoji: "💅"
type: "tech"
topics: ["GAS", "GoogleAppsScript", "美容サロン", "シフト管理", "自動化"]
published: true
---

## はじめに

美容サロンのシフト管理は、スタッフへの個別連絡・確認・修正対応と手間が多く、オーナーや店長の時間を大きく奪います。LINEやSMSで一人ずつ送るのは抜け漏れの原因にもなります。

Google Apps Script（GAS）を使えば、スプレッドシートでシフトを確定した瞬間にスタッフへ自動通知を送ることができます。この記事では、スタッフ情報と通知先（メールアドレス）をスプレッドシートで管理し、シフト確定のフラグを立てるだけで自動メール通知が飛ぶ仕組みを解説します。

---

## 完成イメージ

シフト表の「確定」列にチェックを入れると、該当スタッフのメールアドレスに以下のような通知が自動送信されます。

```
件名: 【シフト確定】来週のシフトをお知らせします

田中 さま

来週のシフトが確定しました。

勤務日: 2026/03/23（月）
開始時刻: 10:00
終了時刻: 18:00

ご不明な点はオーナーまでご連絡ください。
```

スタッフ全員に個別でメールが届き、シフト管理の連絡工数がゼロになります。

---

## 実装手順

### ステップ1: スプレッドシートの準備

以下の列構成でシフト表を作成します。シート名は「シフト表」とします。

| A列 | B列 | C列 | D列 | E列 | F列 |
|-----|-----|-----|-----|-----|-----|
| スタッフ名 | メールアドレス | 勤務日 | 開始時刻 | 終了時刻 | 確定（TRUE/FALSE） |
| 田中 | tanaka@example.com | 2026/03/23 | 10:00 | 18:00 | FALSE |

F列の「確定」がチェックボックス（TRUE）になった行を検知して通知します。チェックボックスはメニューの「挿入」→「チェックボックス」で挿入できます。

### ステップ2: GASエディタを開く

1. スプレッドシートのメニューから「拡張機能」→「Apps Script」を選択
2. デフォルトのコード（`myFunction`）を削除して、以下のコードを貼り付ける

### ステップ3: シフト確定通知のコードを追加する

```javascript
// シフト確定列（F列 = 6列目）の編集を検知して通知する
function onShiftConfirmed(e) {
  var sheet = e.range.getSheet();
  var sheetName = sheet.getName();

  // 「シフト表」シート以外は無視
  if (sheetName !== "シフト表") return;

  var row = e.range.getRow();
  var col = e.range.getColumn();

  // F列（6列目）かつ2行目以降のみ処理
  if (col !== 6 || row < 2) return;

  var isConfirmed = e.range.getValue();

  // チェックが外れた場合は通知しない
  if (isConfirmed !== true) return;

  // スタッフ情報を取得
  var staffName  = sheet.getRange(row, 1).getValue(); // A列: スタッフ名
  var staffEmail = sheet.getRange(row, 2).getValue(); // B列: メールアドレス
  var workDate   = sheet.getRange(row, 3).getValue(); // C列: 勤務日
  var startTime  = sheet.getRange(row, 4).getValue(); // D列: 開始時刻
  var endTime    = sheet.getRange(row, 5).getValue(); // E列: 終了時刻

  // 日付フォーマット
  var formattedDate = Utilities.formatDate(
    new Date(workDate), "Asia/Tokyo", "yyyy/MM/dd（E）"
  );

  // メール本文を作成
  var subject = "【シフト確定】来週のシフトをお知らせします";
  var body = staffName + " さま\n\n"
    + "来週のシフトが確定しました。\n\n"
    + "勤務日: " + formattedDate + "\n"
    + "開始時刻: " + startTime + "\n"
    + "終了時刻: " + endTime + "\n\n"
    + "ご不明な点はオーナーまでご連絡ください。";

  // メール送信
  if (staffEmail) {
    GmailApp.sendEmail(staffEmail, subject, body);
  }
}
```

**コードのポイント:**

- `sheetName !== "シフト表"` でシート名を限定し、他シートの編集で誤作動しません
- `isConfirmed !== true` の条件で、チェックを外した時には通知しません
- `Utilities.formatDate` で曜日付き日付（例: 2026/03/23（月））を生成しています

### ステップ4: インストーラブルトリガーを設定する

`GmailApp.sendEmail` は認証が必要なため、単純トリガーではなくインストーラブルトリガーを使います。

1. GASエディタ左メニューの「トリガー」（時計アイコン）を開く
2. 右下の「トリガーを追加」をクリック
3. 以下の設定を行う:
   - 実行する関数: `onShiftConfirmed`
   - イベントのソース: 「スプレッドシートから」
   - イベントの種類: 「編集時」
4. 「保存」をクリック（Googleアカウントの認証画面が出たら許可する）

これでF列のチェックボックスをONにするたびに、対象スタッフへ自動メールが送信されます。

### ステップ5: Slack通知も併用する（応用）

オーナーへのSlack通知を追加する場合、以下のコードを `onShiftConfirmed` 関数の末尾に追加します。

```javascript
  // Slack通知（オーナー確認用）
  var slackWebhookUrl = PropertiesService.getScriptProperties()
    .getProperty("SLACK_WEBHOOK");

  if (slackWebhookUrl) {
    var slackMessage = "[シフト確定] " + staffName + " / " + formattedDate
      + " " + startTime + "〜" + endTime;

    UrlFetchApp.fetch(slackWebhookUrl, {
      method: "post",
      contentType: "application/json",
      payload: JSON.stringify({ text: slackMessage })
    });
  }
```

Slack Webhook URLはスクリプトプロパティに `SLACK_WEBHOOK` キーで登録してください（「プロジェクトの設定」→「スクリプト プロパティ」）。

---

## まとめ

- F列（確定フラグ）の編集をトリガーにすることで、確定操作だけで通知が完結する
- スタッフ名・メールアドレスをシート上で管理するため、スタッフ増減にも柔軟に対応できる
- インストーラブルトリガーを使うことで `GmailApp` の認証問題を回避できる
- Slack通知を追加すればオーナー側のリアルタイム確認も可能

シフト確定の連絡作業を完全に自動化することで、管理業務の負担を大幅に削減できます。

---
## テンプレートを使えば5分で導入できます

この記事の内容をすぐに使いたい方向けに、設定済みのGoogle Sheetsテンプレートを用意しました。
GASコード付き・Instructionsタブ付きで、コピーして設定を変えるだけで動きます。

👉 [Auto-Notify Timesheet Template（Gumroad）](https://inoshu.gumroad.com/l/pknnt)
