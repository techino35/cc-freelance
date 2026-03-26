---
title: "GASとClaude APIで会議の録音→議事録を全自動化した話"
emoji: "📝"
type: "tech"
topics: ["GAS", "Claude", "AI", "自動化", "議事録"]
published: false
---

## やりたいこと

毎週の定例会議、終わった後に「誰が議事録書く？」という空気になりませんか。

自分のチームでは以下の流れを手動でやっていました。

1. Google Meetで会議（自動文字起こし有効）
2. 文字起こしをコピー
3. ChatGPTかClaudeに貼り付けて整形
4. ドキュメントに貼り付けてSlackに共有

この工程を**GAS + Claude API**で完全に自動化しました。

完成後の流れはこうです。

```
Google Meet 終了
    ↓（自動）
文字起こしをスプレッドシートに貼り付け（※ここだけ手動）
    ↓（自動）
GAS が起動 → Claude API に送信
    ↓（自動）
議事録が別シートに生成
    ↓（自動）
Slack に要約 + リンクを通知
```

「文字起こしを貼り付ける」1ステップだけで、あとは全部自動になります。

---

## 全体フロー

```
[Google Meet]
    │
    │ 文字起こし（自動生成）
    ▼
[Googleスプレッドシート]
    │ 「input」シートに文字起こしを貼る
    │
    ▼
[Google Apps Script]
    │
    ├─ 1. inputシートから文字起こし取得
    ├─ 2. Claude API に議事録生成を依頼
    ├─ 3. 結果を「output」シートに書き込み
    └─ 4. Slack Incoming Webhook で通知
```

スプレッドシートは2シート構成です。

- `input` シート：文字起こしを貼る場所（A1セルから）
- `output` シート：生成された議事録が入る場所

---

## GASコード

### スクリプトプロパティの設定

GASエディタの「プロジェクトの設定」→「スクリプトプロパティ」に以下を追加します。

| プロパティ名 | 値 |
|---|---|
| CLAUDE_API_KEY | sk-ant-xxxx（Anthropic Console で発行） |
| SLACK_WEBHOOK_URL | https://hooks.slack.com/services/xxxx |
| SPREADSHEET_ID | スプレッドシートのID |

### メインコード

```javascript
// 議事録自動生成メイン関数
function generateMeetingMinutes() {
  const props = PropertiesService.getScriptProperties();
  const apiKey = props.getProperty('CLAUDE_API_KEY');
  const slackWebhookUrl = props.getProperty('SLACK_WEBHOOK_URL');
  const spreadsheetId = props.getProperty('SPREADSHEET_ID');

  // 文字起こしを取得
  const ss = SpreadsheetApp.openById(spreadsheetId);
  const inputSheet = ss.getSheetByName('input');
  const transcript = inputSheet.getRange('A1').getValue();

  if (!transcript || transcript.trim() === '') {
    Logger.log('文字起こしが入力されていません');
    return;
  }

  // Claude API で議事録生成
  const minutes = callClaudeAPI(apiKey, transcript);
  if (!minutes) {
    Logger.log('Claude API の呼び出しに失敗しました');
    return;
  }

  // output シートに書き込み
  const outputSheet = ss.getSheetByName('output');
  const timestamp = Utilities.formatDate(new Date(), 'Asia/Tokyo', 'yyyy/MM/dd HH:mm');
  outputSheet.clearContents();
  outputSheet.getRange('A1').setValue('生成日時: ' + timestamp);
  outputSheet.getRange('A2').setValue(minutes);

  // Slack 通知
  const summary = extractSummary(minutes);
  const sheetUrl = ss.getUrl() + '#gid=' + outputSheet.getSheetId();
  notifySlack(slackWebhookUrl, summary, sheetUrl, timestamp);

  // input シートをクリア（次回用）
  inputSheet.getRange('A1').clearContent();

  Logger.log('議事録生成完了: ' + timestamp);
}

// Claude API 呼び出し
function callClaudeAPI(apiKey, transcript) {
  const prompt = buildPrompt(transcript);

  const payload = {
    model: 'claude-3-5-sonnet-20241022',
    max_tokens: 2048,
    messages: [
      {
        role: 'user',
        content: prompt
      }
    ]
  };

  const options = {
    method: 'post',
    contentType: 'application/json',
    headers: {
      'x-api-key': apiKey,
      'anthropic-version': '2023-06-01'
    },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };

  try {
    const response = UrlFetchApp.fetch(
      'https://api.anthropic.com/v1/messages',
      options
    );
    const statusCode = response.getResponseCode();

    if (statusCode !== 200) {
      Logger.log('APIエラー: ' + statusCode + ' ' + response.getContentText());
      return null;
    }

    const json = JSON.parse(response.getContentText());
    return json.content[0].text;

  } catch (e) {
    Logger.log('例外発生: ' + e.message);
    return null;
  }
}

// プロンプト設計
function buildPrompt(transcript) {
  return `以下は会議の文字起こしです。議事録を作成してください。

【文字起こし】
${transcript}

【出力形式】
以下の形式で出力してください。

## 会議概要
（50字以内で会議のテーマを一言で）

## 参加者
（文字起こしから推定できる参加者名）

## 決定事項
- （箇条書きで）

## アクションアイテム
| 担当者 | タスク | 期限 |
|---|---|---|
| （担当者名） | （タスク内容） | （言及があれば） |

## 議論のポイント
（重要な議論や背景を200字以内で）

## 次回確認事項
- （次回MTGで確認すべき点）

【注意事項】
- 文字起こしに含まれない情報は補完しないこと
- 担当者や期限が不明な場合は「未定」と記載
- 敬語を使わずシンプルに書くこと`;
}

// 要約の抽出（Slack通知用）
function extractSummary(minutes) {
  const lines = minutes.split('\n');
  const summaryLines = [];
  let inDecisions = false;
  let inActions = false;

  for (const line of lines) {
    if (line.includes('## 会議概要')) {
      inDecisions = false;
      inActions = false;
    } else if (line.includes('## 決定事項')) {
      summaryLines.push('*決定事項*');
      inDecisions = true;
      inActions = false;
    } else if (line.includes('## アクションアイテム')) {
      summaryLines.push('\n*アクションアイテム*');
      inDecisions = false;
      inActions = true;
    } else if (line.includes('## 議論') || line.includes('## 次回')) {
      break;
    } else if ((inDecisions || inActions) && line.trim().startsWith('-')) {
      summaryLines.push(line.trim());
    }
  }

  return summaryLines.join('\n');
}

// Slack 通知
function notifySlack(webhookUrl, summary, sheetUrl, timestamp) {
  const message = {
    text: ':memo: 議事録が生成されました（' + timestamp + '）',
    attachments: [
      {
        color: '#36a64f',
        text: summary,
        footer: '<' + sheetUrl + '|スプレッドシートで全文を確認>'
      }
    ]
  };

  const options = {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify(message),
    muteHttpExceptions: true
  };

  try {
    const response = UrlFetchApp.fetch(webhookUrl, options);
    Logger.log('Slack通知: ' + response.getResponseCode());
  } catch (e) {
    Logger.log('Slack通知エラー: ' + e.message);
  }
}
```

---

## ポイント解説：プロンプト設計のコツ

### 1. 出力形式を表で指定する

「アクションアイテムをMarkdownの表形式で出力」と明示することで、スプレッドシートへの転記が楽になります。Claude はフォーマット指定に忠実に従います。

### 2. 「補完しないこと」を明示する

文字起こしには欠けている情報が多いです。「担当者が言及されていない場合は補完しないこと」と書かないと、Claude が架空の担当者名を作ることがあります。ハルシネーション対策として必須です。

### 3. モデルは claude-3-5-sonnet を使う

コストと精度のバランスが最も良いです。議事録1件あたりのトークン数は入力2000〜4000、出力1000〜1500程度なので、費用は1回あたり0.5〜1円前後です。

### 4. 文字起こしの前処理

Google Meet の文字起こしには話者名がタイムスタンプ付きで入ります。そのまま送っても Claude は処理できますが、不要な情報（「ん〜」「あー」等のフィラー）が多い場合は以下で簡単に除去できます。

```javascript
function cleanTranscript(raw) {
  return raw
    .replace(/\d{1,2}:\d{2}/g, '')       // タイムスタンプ除去
    .replace(/えーと|あの|えっと|ん〜/g, '') // フィラー除去
    .replace(/\n{3,}/g, '\n\n')            // 空行の整理
    .trim();
}
```

`generateMeetingMinutes` の文字起こし取得後に `cleanTranscript(transcript)` を挟むだけで精度が上がります。

### 5. トリガーの設定

スプレッドシートの `input` シートが編集されたタイミングで自動起動させるなら、`onEdit` トリガーを使います。ただし大量テキストの貼り付けで複数回起動する場合があるため、A1セルに値が存在するときのみ実行するガード節を入れています（コード冒頭の `if (!transcript)` の部分）。

---

## まとめ

- Google Meetの文字起こしを貼るだけで議事録が自動生成される
- Claude のプロンプトは出力形式の指定とハルシネーション対策が肝
- Slack 通知まで自動化することで、参加者全員に即座に共有できる

このような**GAS業務自動化の実践的なテンプレート集**をGumroadで販売しています。議事録テンプレートのほか、日報自動集計・経費申請通知・スプレッドシートバックアップなど10種類以上のすぐに使えるコードが入っています。

https://inoshu.gumroad.com/l/etlfbf
