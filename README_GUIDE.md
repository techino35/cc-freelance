# cc-freelance 一日の使い方ガイド

## 概要
「社長の指示だけで会社が動く」フリーランス管理システム。
Claude Codeを起動して話しかけるだけで、各部署が自律的に動く。

---

## 朝（起床後・作業開始時）

### 1. Claude Codeを起動
```powershell
$env:PATH += ";C:\Users\inoue\.local\bin"
claude
```

### 2. 朝のダッシュボードを表示
```
おはよう
```

**secretaryが自動で以下を実行：**
- 今日のTODO確認
- 最新の経営方針（decisions/）を読み込み
- 未処理の引き継ぎ（handoff/）を確認
- 返信待ち案件の状況確認
- 今月の収入サマリー
- Gmailの重要メール確認
- クラウドワークスの新着案件スコアリング（7点以上のみ表示）

**出力例：**
```
【2026-03-18 朝のサマリー】
収入: ¥2,000 / ¥50,000目標（達成率4%）
最新方針: Upworkは$500以上のみ・固定報酬型を標準化
未処理引き継ぎ: 0件
返信待ち案件: 2件
クラウドワークス注目案件: [スコア8点] Python自動化ツール開発
未読メール（重要）: 岸本さんから契約書
今日の優先タスク:
  1. 岸本さん契約書を確認・署名
  2. OliveKK橋本さんへの返信
  3. Upwork案件に応募
```

---

## 午前（案件対応・営業）

### 案件スコアリング・提案書作成
```
以下の案件をスコアリングして提案書を作って。
[案件のURL or 詳細を貼り付け]
```

**sales-agentが自動で：**
- 10点スコアリングを実施
- 7点未満なら見送り理由を報告
- 7点以上ならパーソナライズされた提案書を生成
- .freelance/sales/proposals/ に保存

### 意思決定を記録
```
Upworkのこの案件は見送りにする。理由はRubyが必須だから。
```
→ secretaryが .freelance/decisions/ に自動記録
→ 次回から全エージェントがこの方針を参照

---

## 午後（開発・クライアント対応）

### 開発タスクの実装
```
OliveKKのGAS通知機能を実装して
```

**engineer-agentが自動で：**
1. プランを提示（承認を求める）
2. 承認後に実装
3. 完了後に引き継ぎファイルを作成

### クライアントへの連絡文作成
```
橋本さんへの進捗報告文を作って
```

**client-agentが自動で：**
- 現在の進捗状況を確認
- 丁寧な報告文を生成
- コミュニケーション履歴に記録

### 稼働時間を記録
```
OliveKKの稼働を1.5時間記録して
```

**client-agentが自動で：**
- .freelance/clients/active/ に記録
- GAS DBの収入台帳に登録

---

## 受注時（案件が決まったとき）

```
岸本さんのec-sync案件を受注した。金額30万円。
```

**以下が自動で実行される：**
1. Google Driveにクライアントフォルダ作成
   ```
   cc-freelance/clients/岸本さん/ec-sync/
   ├── 01_要件定義/
   ├── 02_設計/
   ├── 03_納品物/
   └── 04_請求書/
   ```
2. GitHubにプライベートリポジトリ作成
3. GAS DBの収入台帳に登録
4. 案件パイプラインのステータスを「受注」に更新
5. 意思決定ログに記録

---

## 夕方（振り返り）

### 今日の振り返りを記録
```
今日の振り返りを記録して。
良: Upwork提案でmomo-dropshipの実績を前面に出したら返信が来た
改: 稼働記録を後回しにしがち
次回試すこと: 作業完了直後に稼働時間を記録する習慣をつける
```

→ .freelance/retro/ に保存
→ 翌週の週次レビューで参照される

---

## 週次（毎週月曜）

### 週次レビューを生成
```
週次レビューを生成して
```

**secretaryが自動で：**
- GAS DBから今週の収入・案件状況を集計
- 直近の振り返りメモリ（retro/）を参照
- 改善パターンを抽出
- 来週の優先アクションを提案

**出力フォーマット：**
```
【2026-03-18 週次レビュー】

■ 今週の稼働・収入実績
■ 進行中タスク状況
■ 応募中案件の返信状況
■ 今週の優先アクション3つ
■ 保留・懸念事項
```

---

## 月末（月次締め）

### 月次集計を実行
```
今月の締めをやって
```

**以下が自動で実行される：**
1. GAS DBから当月の稼働・収入データを集計
2. クライアント別の請求額を計算
3. 月次収入レポートを生成
4. 目標との差分を算出
5. 来月の優先アクションを提案

---

## パッシブ収入（自動稼働中）

### Gumroad売上確認
```
パッシブ収入を確認して
```
→ finance-agentがGumroad売上 + ブログ収益を集計して報告

### WordPress自動投稿
```
cd C:\Users\inoue\Downloads\passive-income\phase3
start_poster.bat をダブルクリック
```
→ 1日3記事（8:00 / 14:00 / 20:00 UTC）自動投稿
→ 100記事完了で自動停止（2026-03-19時点 1/100完了）

### 出品済みGumroad商品

| 商品名 | 価格 | URL |
|--------|------|-----|
| Auto-Notify Timesheet | $19 | https://inoshu.gumroad.com/l/pknnt |
| Monthly Report Generator | $24 | https://inoshu.gumroad.com/l/cplqvv |
| Google Maps Scraper | $29 | https://inoshu.gumroad.com/l/xibqmq |
| Freelancer Income Tracker | $15 | https://inoshu.gumroad.com/l/xslxh |
| Form-to-Workflow Kit | $19 | https://inoshu.gumroad.com/l/yqvqc |

### 公開済みZenn記事

| 記事 | URL |
|------|-----|
| GAS通知自動化 | https://zenn.dev/ino38/articles/gas-notification-google-chat-slack |
| GAS PDF月次レポート | https://zenn.dev/ino38/articles/gas-pdf-monthly-report |
| cc-freelance全記録 | https://zenn.dev/ino38/articles/claude-code-freelance-virtual-company |

### 新しいGumroad商品を出品したいとき
```
Gumroadに新商品を出品したい。[商品の概要]
```
→ marketing-agentが英語説明文・チェックリストを生成

---

## 戦略相談（いつでも）

### 事業戦略の分析
```
今月の戦略を分析して。収入目標に対してどう動くべきか教えて。
```

**strategy-agentが自動で：**
- GAS DBから収入・パイプラインを分析
- ギャップ分析を実施
- 優先度付きのアクションプランを提案

---

## 新規プロジェクトチームの編成

### 複雑な案件が来たとき
```
/create-team
```

→ 対話形式でプロジェクト情報を入力
→ 案件特化のエージェントチームが自動生成

---

## MCP連携（エージェントが使えるツール）

| MCP | 用途 |
|-----|------|
| Gmail | メール確認・下書き作成 |
| Google Calendar | 予定確認・スケジュール管理 |
| GitHub | リポジトリ管理・Issue作成 |

---

## ファイル構造（参考）

```
.freelance/
├── secretary/
│   └── todos/          # 日次TODO
├── sales/
│   ├── pipeline/       # 案件パイプライン
│   └── proposals/      # 提案書
├── clients/
│   └── active/         # クライアント別稼働記録
├── engineering/
│   ├── tasks/          # 開発タスク
│   └── debug-log/      # デバッグログ
├── finance/
│   ├── income/         # 月次収入記録
│   ├── invoices/       # 請求書
│   └── passive/        # パッシブ収入記録（gumroad/blog）
├── decisions/          # 意思決定ログ
├── handoff/            # 部署間引き継ぎ
└── retro/              # 振り返りメモリ
```

---

## エージェント一覧

| エージェント | 起動ワード例 |
|------------|------------|
| secretary | 「おはよう」「今日やること」「ダッシュボード」 |
| sales-agent | 「案件応募」「提案書」「スコアリング」 |
| client-agent | 「OliveKK」「稼働記録」「クライアント連絡」 |
| engineer-agent | 「実装」「GAS」「Python」「デバッグ」 |
| finance-agent | 「収入」「請求」「月次集計」 |
| research-agent | 「調べて」「技術調査」「案件リサーチ」 |
| marketing-agent | 「SNS投稿」「ストア説明文」「LP」「Gumroad」「商品説明」 |
| strategy-agent | 「戦略」「次の一手」「事業プラン」 |

---

*最終更新: 2026-03-19*
