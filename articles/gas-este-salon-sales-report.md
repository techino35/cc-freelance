---
title: "エステサロンの売上日報をGoogleスプレッドシートで自動集計・レポート送信する方法"
emoji: "📊"
type: "tech"
topics: ["GAS", "GoogleAppsScript", "エステサロン", "売上管理", "自動化"]
published: true
---

## はじめに

エステサロンの日次売上管理は、レジ締め後にExcelやメモに手入力し、月末にまとめて集計するサロンも多いです。この方法では月次の数字が出るまでに時間がかかり、早期の経営判断が難しくなります。

Google Apps Script（GAS）を使えば、スプレッドシートに日々の売上を入力するだけで、毎日の集計・月次レポートの自動生成・メール送信まで自動化できます。この記事では、エステサロンの実務に沿ったシート設計からGASコードの実装まで解説します。

---

## 完成イメージ

毎日スプレッドシートの「日次売上」シートに入力すると、毎月末日の夜に以下のようなレポートがオーナーのメールに自動送信されます。

```
件名: 【月次売上レポート】2026年3月分

2026年3月の売上レポートです。

---
総売上: ¥1,245,000
施術件数: 87件
客単価: ¥14,310

メニュー別売上:
  フェイシャル: ¥480,000
  ボディ: ¥390,000
  脱毛: ¥375,000
---

詳細はスプレッドシートでご確認ください。
```

月末に手動でExcelを集計する作業がゼロになります。

---

## 実装手順

### ステップ1: スプレッドシートの準備

2枚のシートを用意します。

**「日次売上」シート（毎日入力するシート）:**

| A列 | B列 | C列 | D列 |
|-----|-----|-----|-----|
| 日付 | メニュー | 売上金額 | 施術件数 |
| 2026/03/01 | フェイシャル | 50000 | 4 |
| 2026/03/01 | ボディ | 30000 | 2 |

**「月次サマリー」シート（自動で書き込まれるシート）:**

GASが月次集計結果を自動で書き込みます。手動入力は不要です。

### ステップ2: GASエディタを開く

1. スプレッドシートのメニューから「拡張機能」→「Apps Script」を選択
2. デフォルトのコード（`myFunction`）を削除して、以下のコードを貼り付ける

### ステップ3: 月次集計・レポート送信コードを追加する

```javascript
// 月次売上を集計してメールで送信する
function sendMonthlyReport() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var dailySheet   = ss.getSheetByName("日次売上");
  var summarySheet = ss.getSheetByName("月次サマリー");

  // 先月の年月を取得（月末実行時に先月分を集計する場合は -1）
  var today     = new Date();
  var targetYear  = today.getFullYear();
  var targetMonth = today.getMonth() + 1; // 当月を集計する場合はそのまま使用

  // 日次売上データを全行取得（2行目以降がデータ行）
  var lastRow = dailySheet.getLastRow();
  if (lastRow < 2) {
    Logger.log("売上データがありません");
    return;
  }
  var salesData = dailySheet.getRange(2, 1, lastRow - 1, 4).getValues();

  // 当月データのフィルタリング＆集計
  var totalSales   = 0;
  var totalCount   = 0;
  var menuTotals   = {}; // メニュー別集計

  salesData.forEach(function(row) {
    var date         = new Date(row[0]); // A列: 日付
    var menuName     = row[1];           // B列: メニュー名
    var saleAmount   = Number(row[2]);   // C列: 売上金額
    var treatCount   = Number(row[3]);   // D列: 施術件数

    // 当月のデータのみ処理
    if (date.getFullYear() !== targetYear || date.getMonth() + 1 !== targetMonth) return;

    totalSales += saleAmount;
    totalCount += treatCount;

    // メニュー別集計
    if (!menuTotals[menuName]) {
      menuTotals[menuName] = 0;
    }
    menuTotals[menuName] += saleAmount;
  });

  // 客単価計算（ゼロ除算防止）
  var avgSale = totalCount > 0 ? Math.round(totalSales / totalCount) : 0;

  // 月次サマリーシートに書き込み
  var reportRow = [
    targetYear + "/" + targetMonth,
    totalSales,
    totalCount,
    avgSale
  ];
  summarySheet.appendRow(reportRow);

  // メニュー別集計テキストを作成
  var menuText = "";
  Object.keys(menuTotals).forEach(function(menu) {
    menuText += "  " + menu + ": ¥" + menuTotals[menu].toLocaleString() + "\n";
  });

  // レポートメール本文を作成
  var subject = "【月次売上レポート】" + targetYear + "年" + targetMonth + "月分";
  var body = targetYear + "年" + targetMonth + "月の売上レポートです。\n\n"
    + "---\n"
    + "総売上: ¥" + totalSales.toLocaleString() + "\n"
    + "施術件数: " + totalCount + "件\n"
    + "客単価: ¥" + avgSale.toLocaleString() + "\n\n"
    + "メニュー別売上:\n"
    + menuText
    + "---\n\n"
    + "詳細はスプレッドシートでご確認ください。\n"
    + ss.getUrl();

  // オーナーのメールアドレスをスクリプトプロパティから取得
  var ownerEmail = PropertiesService.getScriptProperties()
    .getProperty("OWNER_EMAIL");

  if (ownerEmail) {
    GmailApp.sendEmail(ownerEmail, subject, body);
    Logger.log("月次レポートを送信しました: " + ownerEmail);
  } else {
    Logger.log("OWNER_EMAIL が設定されていません");
  }
}
```

**コードのポイント:**

- `menuTotals` オブジェクトでメニュー別に動的に集計するため、メニュー追加・削除に自動で対応します
- `PropertiesService` でメールアドレスを管理し、コードへの直書きを避けています
- `ss.getUrl()` でスプレッドシートのURLをレポートに自動添付するため、受信後すぐに詳細を確認できます

### ステップ4: オーナーメールアドレスを設定する

1. GASエディタ左メニュー「プロジェクトの設定」（歯車アイコン）を開く
2. 「スクリプト プロパティ」で「プロパティを追加」
3. プロパティ名: `OWNER_EMAIL`、値: オーナーのGmailアドレスを入力
4. 「スクリプト プロパティを保存」

### ステップ5: 月末自動実行トリガーを設定する

1. GASエディタ左メニューの「トリガー」（時計アイコン）を開く
2. 右下の「トリガーを追加」をクリック
3. 以下の設定を行う:
   - 実行する関数: `sendMonthlyReport`
   - イベントのソース: 「時間主導型」
   - 時間ベースのトリガーのタイプ: 「月ベースのタイマー」
   - 日: 「月の最終日」
   - 時刻: 「午後10時〜11時」
4. 「保存」をクリック

これで毎月末日の夜にレポートが自動送信されます。

---

## 応用: 週次レポートも送る

週次で売上を把握したい場合は、同じ関数を週次トリガーで実行するだけです。トリガーの設定で「週ベースのタイマー」→「毎週日曜日」を選択してください。集計期間の絞り込みロジックを週単位に変更する必要があります。

```javascript
// 週次集計の場合（過去7日分）
var oneWeekAgo = new Date();
oneWeekAgo.setDate(oneWeekAgo.getDate() - 7);
// date >= oneWeekAgo の条件でフィルタリングする
```

---

## まとめ

- 日次売上シートへの入力だけで月次集計が完全自動化される
- メニュー別集計を動的に行うため、メニュー変更にも柔軟に対応できる
- `PropertiesService` でメールアドレスを安全に管理できる
- 月末トリガーにより、手動操作なしでレポートが届く

売上集計の手作業をなくすことで、数字の把握が早くなり、経営判断のスピードが上がります。

---
## テンプレートを使えば5分で導入できます

この記事の内容をすぐに使いたい方向けに、設定済みのGoogle Sheetsテンプレートを用意しました。
GASコード付き・Instructionsタブ付きで、コピーして設定を変えるだけで動きます。

👉 [Monthly Report Generator（Gumroad）](https://inoshu.gumroad.com/l/cplqvv)
