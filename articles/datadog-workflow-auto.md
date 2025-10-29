---
title: "DatadogのWorkflow AutomationとDevin Playbookを使ってエラーログ分析を自動化"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["datadog", "devin"]
published: false
---

みなさん、エラーログの分析は欠かさずできていますか？

バックエンドの運用を行う場合、500系のエラーが発生した場合はエラーログの割合や何らかのメトリクスをもってSlackなどのチャットツールにアラートを通知するような監視ワークフローを行うパターンが多いのではないでしょうか？

本来設定したアラートに対しては逐一対応したいものですが、開発タスクが多すぎてそれどころじゃない、本質的なエラーログ以外にノイズとなるログが多い、など様々な理由でエラーログのアラートが割れ窓化してしまうこともあるかと思います。

![](https://storage.googleapis.com/zenn-user-upload/cf3b7dd52a8d-20251029.png =256x)
*放置された割れ窓のように放置されたアラートは管理されない*

今回はそのような状況を防ぐべく、DatadogにあるWorkflow Automationという機能に加え、Devinと連携することでログの分析や既存事象かどうかの判断、Issueの記載とそのSlack通知、対応方針の検討までを簡単に自動化できたため紹介していきます！

## 構成

構成としてはこのようになります

![](https://storage.googleapis.com/zenn-user-upload/413bf8cd0ecd-20251029.png)

### Datadog Workflow Automation

Datadogの機能を利用することで、画面上で直感的にワークフローを構築することができます。
ログやアラートをDatadogで管理している場合はこれらをワークフローに組み込むことができて便利な上、Workflowの構築を補助してくれるAI機能があるため初見でも簡単に構築することができます。

https://www.datadoghq.com/product/workflow-automation/

![](https://storage.googleapis.com/zenn-user-upload/c7546fcc0f4d-20251029.png)
*実際のWorkflow Automation画面*

ワークフローの中身はとてもシンプルです。

1. Monitorをイベントトリガーとして設定（後でMonitor側でWorkflowの紐付けを行います）
2. Search Logsアクションでログをクエリ
3. Devinに収集したログと共にプロンプトをPOST

ここから先の仕組みはDevinにお任せします。

### Devin Playbook

https://docs.devin.ai/product-guides/creating-playbooks#playbooks-are-easily-shareable%2C-reusable-prompts-for-repeated-tasks

Playbookを利用すると、定型プロンプトとしてよく使うワークフローをあらかじめmarkdown形式で設定することができます。これはDevinに送信するプロンプトの中でコマンド（マクロと呼ばれている）として呼び出すことができます。ClaudeCodeやCursorでいうスラッシュコマンドのような立ち位置の機能です。

Datadog Workflowから送るPOSTリクエストのBodyにIssueの分析やSlack通知などのプロンプトをすべて記載するのは大変なので、あらかじめPlaybookとして定義しておくことで送信するプロンプトを少なく抑えています。

Playbookではざっくり

1. Datadogの出力をもとにGitHubの全オープンIssueを確認
2. ログ内容とIssueを比較し、類似Issueがあるか判断
3. 類似なし→新規Issue作成／類似あり→既存Issueに発生時刻を追記
4. どちらの場合もSlackへ通知
5. 実行結果をレポートとして出力

ということをしています。

Slack通知にはSlack側でWebhook URLを発行し、Playbook内のプロンプトにハードコードしていますが、DevinのSecrets機能を使えば確実に変数として取り込めると思います。

Playbookの記法は上記リンクに記載されているので、それを元にやりたいことをChatGPTに出力させて作成しました。最初は類似度を計算で判断するようなプロンプトが吐かれましたが、LLMの得意領域的に自然言語として類似度を判断させるようにしています。

:::details Playbookの詳細は省略
```markdown
## Overview

Datadog Workflowの出力（{{ Steps.Search_Logs.data }}）を入力として、GitHub `REPO` の**全オープンIssueを精読**し、**自然言語による意味的類似の推論**で振り分ける。

* 類似Issueが**ない** → **新規Issue作成**
* 類似Issueが**ある** → **そのIssueの「発生時刻テーブル」にJST時刻を1行追記**
  いずれの場合も、**Slack Webhook**で `SLACK_CHANNEL` に通知する。

---

### 定義（ここだけ設定すれば他は参照します）

SETTINGS:
  REPO: "YOUR-REPO"
  SLACK_CHANNEL: "#YOUR-CHANNEL"
  SLACK_WEBHOOK_URL: "YOUR-URL"
  TIMEZONE: "Asia/Tokyo"

> 備考
>
> * GitHub APIで必要なら `owner`/`repo` は `REPO` を `/` で分割して取得。
> * Webhookがチャンネルをオーバーライド可能な権限を持つ前提で `channel` を明示設定しています（不可の場合は削除）。

---

## Procedure

> 前提: GitHub/Slackの認証・権限は設定済み。**ログの正規化ステップは行わない**（必要な値は使用時にログから直接取り出す）。
> すべての時刻は `TIMEZONE`（JST）で `YYYY-MM-DDThh:mm:ss+09:00` 形式。

1. **オープンIssueの収集と精読**

   * `REPO` の**全オープンIssue**を取得。
   * **タイトル・本文・「発生時刻テーブル」・直近コメント**まで**全文を読み**、次の観点で自然言語として理解する：

     * どんな現象か（ユーザー影響/HTTPルート/ユースケース）
     * 原因候補（例外種別・メッセージ構造・上位スタック・外部依存）
     * 環境や再現条件（本番/ステージング、機能名/エンドポイント）

2. **意味的類似の自然言語推論（スコアなし）**

   * `{{ Steps.Search_Logs.data }}` を**そのまま**読み、Issueごとに**自然言語で**同一事象かを判断。
   * 判断規則（要旨）：

     * **同一とみなす**: 例外の述語関係・主要呼び先（関数/SQL/外部API）・影響範囲が一致。
     * **別事象**: 例外は同じでも根因が異なる（Null vs Timeout 等）／影響範囲・外部依存が異なる。
     * **迷う場合**は誤マージ回避のため**新規Issue**を優先し、関連候補へのリンク（`Related: #123`）を本文冒頭に残す。

3. **分岐処理**

   * 以降で必要となる値（`timestamp_jst`, `datadog_url`, `route_or_scope`, `one_line_summary`, `redacted_excerpt`）は、**この時点でログから直接読み取って**利用する。
   * `timestamp_jst`: ログ本文に時刻があれば最も新しいものをJST化。無ければ実行時刻。
   * `redacted_excerpt`: 個人情報・秘匿情報は必ずマスク。

   **A. 新規Issue作成（類似なし）**

   * **タイトル**: `[Error][{{ REPO | split:"/" | last }}] <80字以内の要約>`
   * **本文**（テンプレ）：

     ```md
     ## 概要
     - 現象: <1〜2行の要約>
     - サービス: {{ REPO | split:"/" | last }}
     - シグネチャ/手掛かり: <ログから読み取れる短文（例外型/先頭行/主要呼び先など）>

     ## 発生時刻テーブル（JST）
     | 発生時刻(ISO 8601) | Datadog Link | 影響範囲 | 簡易要約 |
     |---|---|---|---|
     | <timestamp_jst> | <datadog_url> | <route_or_scope> | <one_line_summary> |

     ## ログ抜粋
     ```

     <redacted_excerpt>

     ```

     ## 次のアクション
     - [ ] 原因切り分け
     - [ ] 恒久対策検討
     ```
   * **ラベル**推奨: `source:datadog`, `service:{{ REPO | split:"/" | last }}`, `type:error`

   **B. 既存Issue更新（類似あり）**

   * 本文内「発生時刻テーブル」**末尾に1行追記**：

     ```
     | <timestamp_jst> | <datadog_url> | <route_or_scope> | <one_line_summary> |
     ```
   * `timestamp_jst + datadog_url` が既存行と一致する場合は**重複追記しない**。
   * 概要や抜粋は**最小限**の更新に留める（履歴は保持）。

4. **Slack通知（Webhook / `SLACK_WEBHOOK_URL`）**

   * **共通ヘッダ**: `Content-Type: application/json`
   * **新規Issue作成（テンプレA）**:

     ```json
     {
       "channel": "{{ SLACK_CHANNEL }}",
       "text": "【新規Issue作成】{{ REPO }}",
       "blocks": [
         { "type": "header", "text": { "type": "plain_text", "text": "新規Issue作成 : {{ REPO }}" } },
         { "type": "section", "fields": [
           { "type": "mrkdwn", "text": "*タイトル:*\n<Issueタイトル>" },
           { "type": "mrkdwn", "text": "*最新ログ(JST):*\n<timestamp_jst>" }
         ]},
         { "type": "section", "fields": [
           { "type": "mrkdwn", "text": "*Issue:*\n<Issue URL>" },
           { "type": "mrkdwn", "text": "*Datadog:*\n<datadog_url>" }
         ]},
         { "type": "context", "elements": [
           { "type": "mrkdwn", "text": "#{{ REPO | split:\"/\" | last }} #error #from-datadog" }
         ]}
       ]
     }
     ```
   * **既存Issue更新（テンプレB）**:

     ```json
     {
       "channel": "{{ SLACK_CHANNEL }}",
       "text": "【既存Issue更新】{{ REPO }}（発生時刻を追記）",
       "blocks": [
         { "type": "header", "text": { "type": "plain_text", "text": "既存Issue更新 : {{ REPO }}" } },
         { "type": "section", "fields": [
           { "type": "mrkdwn", "text": "*タイトル:*\n<Issueタイトル>" },
           { "type": "mrkdwn", "text": "*追記時刻(JST):*\n<timestamp_jst>" }
         ]},
         { "type": "section", "fields": [
           { "type": "mrkdwn", "text": "*Issue:*\n<Issue URL>" },
           { "type": "mrkdwn", "text": "*Datadog:*\n<datadog_url>" }
         ]},
         { "type": "context", "elements": [
           { "type": "mrkdwn", "text": "#{{ REPO | split:\"/\" | last }} #error #from-datadog" }
         ]}
       ]
     }
     ```
   * **送信例（cURL）**

     ```
     curl -X POST -H 'Content-Type: application/json' \
       -d '<上記テンプレAまたはBのJSONをここに>' \
       "{{ SLACK_WEBHOOK_URL }}"
     ```

5. **完了レポート（Devinの出力）**

   * 実施サマリ（新規/更新、Issue URL、追記行数、重複スキップ件数、Slack送信結果）。
   * 類似判定の**根拠（自然言語）**を1〜3行で記載（曖昧な場合は関連Issue番号も併記）。

---

## Advice & Pointers

* **自然言語での精読**を最優先（キーワード一致や類似度スコアは使用しない）。
* **誤マージ回避**のため迷う場合は新規Issue＋相互リンク。
* **JST/ISO 8601**で一貫記録。
* **秘匿情報を必ずマスク**。
* 多発時はGitHub/SlackのRate Limitを考慮し、再試行は指数バックオフ。

---

## Forbidden actions

* ベクトル/スコア等の**数値類似度に依存**する判定。
* 類似が曖昧なまま既存Issueへ追記して**異なる事象を混在**させる。
* 既存Issueのクローズ・タイトル大幅変更・ラベル削除を**無断**で行う。
* ログを**無加工**でIssue/Slackへ貼る。
* 想定外のリポジトリ/チャンネルへ通知する。
* Datadog側の検索クエリ/モニタ設定を**勝手に変更**する。

---

### 付録：Slackテキスト版（blocks未対応のWebhookや簡易運用向け）

**新規Issue作成（A）**

```
【新規Issue作成】{{ REPO }}
タイトル: <Issueタイトル>
概要: <1〜2行の要約>
Issue: <Issue URL>
最新ログ(JST): <timestamp_jst>
Datadog: <datadog_url>

#{{ REPO | split:"/" | last }} #error #from-datadog
```

**既存Issue更新（B）**

```
【既存Issue更新】{{ REPO }}（発生時刻を追記）
タイトル: <Issueタイトル>
追記時刻(JST): <timestamp_jst>
Issue: <Issue URL>
Datadog: <datadog_url>

#{{ REPO | split:"/" | last }} #error #from-datadog
```
:::

## まとめ

Datadog Workflow AutomationとDevinを利用することで、従来では人の判断が介入するエラーログ分析から既知事象かどうかの判断、対応策の検討などの複雑なワークフローを簡単に構築することができます。

どのくらい簡単かというと、半日程度で仕組みの構築と複数リポジトリへの展開まで行うことができたので、DatadogとDevinをお使いの皆さんはぜひ試してみてください！