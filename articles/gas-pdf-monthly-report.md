---
title: "GASでスプレッドシートを月末に自動でPDF化してメール送信する方法"
emoji: "📄"
type: "tech"
topics: ["GAS", "GoogleAppsScript", "自動化", "スプレッドシート"]
published: true
---

## はじめに

月次レポートをスプレッドシートで管理していると、月末のたびに「PDFに書き出して」「メールに添付して」「送信先ごとに件名を変えて...」という作業が発生します。これを毎月手作業でやるのは単純ですが確実に時間を取られます。Google Apps Script（GAS）を使えば、月末に自動でPDF化してメール送信するところまで完全自動化できます。

---

## 完成イメージ

毎月28日の午前9時に、以下のようなメールが自動送信されます。

**件名:**
```
Monthly Report - 2026年03月
```

**本文:**
```
Please find the monthly report attached.
```

**添付ファイル:**
```
Monthly_Report_2026-03.pdf
```

スプレッドシートの「Dashboard」シートがそのままPDF化されて送られてくるため、受信者はスプレッドシートを開かずにレポートを確認できます。

---

## 実装手順

### ステップ1: スプレッドシートにDashboardシートを作る

スプレッドシートに「Dashboard」という名前のシートを作成します。このシートがPDF化の対象になります。

レポートに含めたい情報（売上サマリー、KPI、グラフなど）をこのシートにまとめておきます。印刷範囲を設定しておくと、PDFの出力範囲を制御できます。

**印刷範囲の設定:**
1. DashboardシートでPDF化したい範囲を選択
2. メニュー「表示」→「印刷範囲を設定」で範囲を固定

### ステップ2: GASエディタを開いてコードを追加する

スプレッドシートのメニューから「拡張機能」→「Apps Script」を開き、以下のコードを貼り付けます。

```javascript
function sendMonthlyReport() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("Dashboard");
  var ssId = ss.getId();
  var sheetId = sheet.getSheetId();

  var url = "https://docs.google.com/spreadsheets/d/" + ssId + "/export?format=pdf&gid=" + sheetId + "&portrait=true&fitw=true";
  var token = ScriptApp.getOAuthToken();
  var response = UrlFetchApp.fetch(url, { headers: { Authorization: "Bearer " + token } });
  var pdf = response.getBlob().setName("Monthly_Report_" + Utilities.formatDate(new Date(), "Asia/Tokyo", "yyyy-MM") + ".pdf");

  var email = PropertiesService.getScriptProperties().getProperty("REPORT_EMAIL");
  GmailApp.sendEmail(email, "Monthly Report - " + Utilities.formatDate(new Date(), "Asia/Tokyo", "yyyy年MM月"), "Please find the monthly report attached.", { attachments: [pdf] });
}

function setMonthlyTrigger() {
  ScriptApp.newTrigger("sendMonthlyReport").timeBased().onMonthDay(28).atHour(9).create();
}
```

**コードのポイント:**

- PDF化はGoogleのExport APIを直接呼び出しています。`ScriptApp.getOAuthToken()` でOAuth認証を自動処理するため、追加の認証設定は不要です
- `portrait=true` で縦向き、`fitw=true` で幅を自動フィットするオプションです
- `REPORT_EMAIL` はスクリプトプロパティから取得します。複数の宛先に送る場合はカンマ区切りで指定できます
- `setMonthlyTrigger` 関数は一度だけ実行してトリガーを登録します。毎回実行するものではありません

### ステップ3: スクリプトプロパティを設定する

1. GASエディタの左メニュー「プロジェクトの設定」（歯車アイコン）を開く
2. 「スクリプト プロパティ」で「プロパティを追加」
3. プロパティ名: `REPORT_EMAIL`、値: 送信先メールアドレスを入力
4. 「スクリプト プロパティを保存」

複数宛先の場合:
```
REPORT_EMAIL = tanaka@example.com,yamada@example.com
```

### ステップ4: トリガーを設定する

`setMonthlyTrigger` 関数を一度だけ手動実行してトリガーを登録します。

1. GASエディタ上部の関数選択ドロップダウンで `setMonthlyTrigger` を選択
2. 「実行」ボタンをクリック
3. 認証画面が出たら「許可」を選択

実行後、左メニューの「トリガー」（時計アイコン）を開いて、毎月28日のトリガーが登録されていることを確認します。

**注意:** `setMonthlyTrigger` を複数回実行すると重複したトリガーが作成されます。既存のトリガーはトリガー一覧画面から削除してから再登録してください。

### ステップ5: 動作確認

本番のトリガー設定前に、手動で `sendMonthlyReport` を実行して動作確認します。

1. 関数選択ドロップダウンで `sendMonthlyReport` を選択
2. 「実行」をクリック
3. 指定したメールアドレスにPDF付きメールが届くか確認

---

## 応用: 複数シートをまとめてPDF化する

Dashboardシート以外も含めて1つのPDFにまとめたい場合は、スプレッドシート全体をエクスポートする方法があります。

```javascript
function sendMonthlyReportAllSheets() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var ssId = ss.getId();

  // スプレッドシート全体をPDF化（全シートを含む）
  var url = "https://docs.google.com/spreadsheets/d/" + ssId + "/export?format=pdf&portrait=true&fitw=true&gridlines=false&printtitle=false";
  var token = ScriptApp.getOAuthToken();
  var response = UrlFetchApp.fetch(url, { headers: { Authorization: "Bearer " + token } });
  var pdf = response.getBlob().setName("Monthly_Report_Full_" + Utilities.formatDate(new Date(), "Asia/Tokyo", "yyyy-MM") + ".pdf");

  var email = PropertiesService.getScriptProperties().getProperty("REPORT_EMAIL");
  GmailApp.sendEmail(
    email,
    "Monthly Report (Full) - " + Utilities.formatDate(new Date(), "Asia/Tokyo", "yyyy年MM月"),
    "Please find the monthly report attached.",
    { attachments: [pdf] }
  );
}
```

**URLパラメータの説明:**

| パラメータ | 値 | 意味 |
|-----------|---|------|
| `format` | `pdf` | PDF形式でエクスポート |
| `portrait` | `true/false` | 縦向き/横向き |
| `fitw` | `true` | 幅を自動フィット |
| `gridlines` | `false` | グリッド線を非表示 |
| `printtitle` | `false` | タイトルを非表示 |
| `gid` | シートID | 特定シートのみ（省略で全シート） |

---

## まとめ

- GASのExport APIでスプレッドシートをPDF化できる
- `ScriptApp.getOAuthToken()` でOAuth認証を自動処理できる
- 時間ベーストリガーで毎月自動実行できる
- URLパラメータで出力形式を細かく制御できる

月次レポートの送付作業は一度自動化してしまえば、毎月確実に実行されます。手動作業のミスや抜け漏れを防げるだけでなく、月末の作業負担を大幅に削減できます。

---
## 設定済みテンプレートで今すぐ使えます

👉 [Monthly Report Generator（Gumroad）](https://gumroad.com/l/monthly-report-generator)
