---
title: "美容院の予約リマインダーをGASで自動化する方法【前日メール通知】"
emoji: "✂️"
type: "tech"
topics: ["GAS", "GoogleAppsScript", "美容院", "予約管理", "リマインダー"]
published: true
---

## はじめに

美容院のノーショー（無断キャンセル）は売上ロスに直結します。前日にリマインダーを送るだけで無断キャンセル率が大幅に下がることは多くのサロンが経験していますが、一人ひとりに手動でメールを送るのは現実的ではありません。

Google Apps Script（GAS）を使えば、スプレッドシートで予約を管理しながら、翌日の予約がある顧客へ自動でリマインダーメールを送ることができます。この記事では、予約管理シートの設計からリマインダー送信の自動化まで、実務レベルのGASコードと一緒に解説します。

---

## 完成イメージ

毎朝9時に自動実行され、翌日の予約が入っている顧客全員に以下のリマインダーメールが送信されます。

```
件名: 【予約確認】明日のご来店についてご案内します

佐藤 様

明日のご予約を確認いたしました。

予約日時: 2026/03/24（火）14:00
担当スタイリスト: 鈴木
メニュー: カット＋カラー

ご都合が悪い場合は、お手数ですがご連絡ください。
TEL: 03-XXXX-XXXX

ご来店をお待ちしております。
Hair Salon ○○
```

毎日の連絡作業を自動化しながら、顧客へ丁寧な印象を与えることができます。

---

## 実装手順

### ステップ1: スプレッドシートの準備

「予約管理」という名前のシートを作成し、以下の列構成でデータを管理します。

| A列 | B列 | C列 | D列 | E列 | F列 |
|-----|-----|-----|-----|-----|-----|
| 顧客名 | メールアドレス | 予約日 | 予約時刻 | 担当 | メニュー |
| 佐藤 | sato@example.com | 2026/03/24 | 14:00 | 鈴木 | カット＋カラー |

予約が入るたびにこのシートに1行追加するだけです。過去の予約は残しておいても問題ありません（当日に翌日分だけ抽出して送信します）。

### ステップ2: GASエディタを開く

1. スプレッドシートのメニューから「拡張機能」→「Apps Script」を選択
2. デフォルトのコード（`myFunction`）を削除して、以下のコードを貼り付ける

### ステップ3: リマインダー送信コードを追加する

```javascript
// 翌日の予約がある顧客にリマインダーメールを送信する
function sendReservationReminders() {
  var ss             = SpreadsheetApp.getActiveSpreadsheet();
  var reservationSheet = ss.getSheetByName("予約管理");

  // 翌日の日付を取得
  var tomorrow = new Date();
  tomorrow.setDate(tomorrow.getDate() + 1);
  var tomorrowStr = Utilities.formatDate(tomorrow, "Asia/Tokyo", "yyyy/MM/dd");

  // 予約データを全行取得（2行目以降がデータ行）
  var lastRow = reservationSheet.getLastRow();
  if (lastRow < 2) {
    Logger.log("予約データがありません");
    return;
  }
  var reservations = reservationSheet.getRange(2, 1, lastRow - 1, 6).getValues();

  // サロン情報をスクリプトプロパティから取得
  var props       = PropertiesService.getScriptProperties();
  var salonName   = props.getProperty("SALON_NAME")   || "Hair Salon";
  var salonTel    = props.getProperty("SALON_TEL")    || "";
  var salonEmail  = props.getProperty("SALON_EMAIL")  || "";

  var sentCount = 0;

  reservations.forEach(function(row) {
    var customerName  = row[0]; // A列: 顧客名
    var customerEmail = row[1]; // B列: メールアドレス
    var reservDate    = row[2]; // C列: 予約日
    var reservTime    = row[3]; // D列: 予約時刻
    var stylist       = row[4]; // E列: 担当
    var menuName      = row[5]; // F列: メニュー

    // 予約日と翌日を比較
    var reservDateStr = Utilities.formatDate(
      new Date(reservDate), "Asia/Tokyo", "yyyy/MM/dd"
    );
    if (reservDateStr !== tomorrowStr) return;

    // メールアドレスが未入力の行はスキップ
    if (!customerEmail) return;

    // 予約日時の曜日付きフォーマット
    var formattedDate = Utilities.formatDate(
      new Date(reservDate), "Asia/Tokyo", "yyyy/MM/dd（E）"
    );

    // 時刻フォーマット（Date型の場合と文字列の場合に対応）
    var formattedTime = (reservTime instanceof Date)
      ? Utilities.formatDate(reservTime, "Asia/Tokyo", "HH:mm")
      : String(reservTime);

    // メール本文を作成
    var subject = "【予約確認】明日のご来店についてご案内します";
    var body = customerName + " 様\n\n"
      + "明日のご予約を確認いたしました。\n\n"
      + "予約日時: " + formattedDate + " " + formattedTime + "\n"
      + "担当スタイリスト: " + stylist + "\n"
      + "メニュー: " + menuName + "\n\n"
      + "ご都合が悪い場合は、お手数ですがご連絡ください。\n"
      + (salonTel ? "TEL: " + salonTel + "\n" : "")
      + (salonEmail ? "MAIL: " + salonEmail + "\n" : "")
      + "\nご来店をお待ちしております。\n"
      + salonName;

    // メール送信
    GmailApp.sendEmail(customerEmail, subject, body);
    sentCount++;
    Logger.log("リマインダー送信: " + customerName + " <" + customerEmail + ">");
  });

  Logger.log("送信完了: " + sentCount + "件");
}
```

**コードのポイント:**

- `tomorrowStr` と `reservDateStr` を文字列で比較することで、時刻のズレによる比較ミスを防いでいます
- 時刻がDate型で入っている場合と文字列で入っている場合を両方処理しています（スプレッドシートの入力方法による差異に対応）
- サロン名・電話番号・メールアドレスをスクリプトプロパティで管理し、コード変更なしに設定変更できます

### ステップ4: スクリプトプロパティを設定する

1. GASエディタ左メニュー「プロジェクトの設定」（歯車アイコン）を開く
2. 「スクリプト プロパティ」で以下の3つを追加する:

| プロパティ名 | 値の例 |
|-------------|--------|
| SALON_NAME | Hair Salon ○○ |
| SALON_TEL | 03-XXXX-XXXX |
| SALON_EMAIL | info@salon-example.com |

3. 「スクリプト プロパティを保存」

### ステップ5: 毎朝9時の自動実行トリガーを設定する

1. GASエディタ左メニューの「トリガー」（時計アイコン）を開く
2. 右下の「トリガーを追加」をクリック
3. 以下の設定を行う:
   - 実行する関数: `sendReservationReminders`
   - イベントのソース: 「時間主導型」
   - 時間ベースのトリガーのタイプ: 「日付ベースのタイマー」
   - 時刻: 「午前9時〜10時」
4. 「保存」をクリック（Googleアカウントの認証画面が出たら許可する）

これで毎朝9時に翌日の予約者全員へ自動でリマインダーが送られます。

---

## 応用: 送信済みフラグで二重送信を防ぐ

予約の日付修正などでデータが変わっても二重送信を防ぎたい場合は、G列に「送信済み」フラグを追加します。

```javascript
  // 送信後にG列へフラグを立てる（二重送信防止）
  var flagCol = 7; // G列
  reservations.forEach(function(row, index) {
    // ...（送信処理の後）
    var flagCell = reservationSheet.getRange(index + 2, flagCol);
    if (flagCell.getValue() === "送信済み") return; // スキップ
    // メール送信後
    flagCell.setValue("送信済み");
  });
```

G列に「送信済み」と書かれた行は次回以降スキップされます。

---

## まとめ

- 翌日の日付で予約データをフィルタリングし、対象顧客のみにメールを送る
- `PropertiesService` でサロン情報を一元管理し、コード変更なしに運用できる
- 毎朝9時の自動トリガーで完全にハンズフリーな運用が実現する
- 送信済みフラグを追加することで安全に運用できる

前日リマインダーの自動化で無断キャンセルを減らし、顧客への丁寧な対応を仕組みとして実現できます。

---
## テンプレートを使えば5分で導入できます

この記事の内容をすぐに使いたい方向けに、設定済みのGoogle Sheetsテンプレートを用意しました。
GASコード付き・Instructionsタブ付きで、コピーして設定を変えるだけで動きます。

👉 [Auto-Notify Timesheet Template（Gumroad）](https://inoshu.gumroad.com/l/pknnt)
