# コンポーネント依存関係

---

## 依存関係図

```mermaid
flowchart TB
    C8[C8: Dashboard UI<br/>React SPA]
    C7[C7: Dashboard API<br/>API Gateway + Lambda]
    C1[C1: Auth<br/>Cognito]
    C4[C4: Score Calculator]
    C3[C3: Judgment Engine]
    C6[C6: Action Tracker]
    C2[C2: Slack Connector<br/>Lambda]
    C5[C5: Daily Reporter<br/>EventBridge + Lambda]
    BEDROCK[Amazon Bedrock<br/>Claude]
    DDB[(DynamoDB<br/>Unit別テーブル)]
    SLACK_API[Slack API<br/>Bot Token]
    SLACK_WH[Slack Webhook<br/>DM送信]

    C8 -->|REST API| C7
    C7 -->|トークン検証| C1
    C7 --> C3
    C7 --> C4
    C7 --> C6
    C7 --> C2

    C5 --> C3
    C5 --> C4
    C5 --> C6
    C5 -->|DM送信| SLACK_WH
    C5 --> BEDROCK

    C3 --> C2
    C3 --> BEDROCK
    C3 --> DDB

    C4 --> DDB
    C6 --> C2
    C6 --> DDB
    C2 --> SLACK_API
    C2 --> DDB

    style C8 fill:#E8F5E9,stroke:#2E7D32
    style C7 fill:#F3E5F5,stroke:#6A1B9A
    style C1 fill:#FFF3E0,stroke:#E65100
    style C3 fill:#FCE4EC,stroke:#880E4F
    style C4 fill:#F1F8E9,stroke:#33691E
    style C5 fill:#FFF8E1,stroke:#F57F17
    style C6 fill:#EDE7F6,stroke:#4527A0
    style C2 fill:#E0F7FA,stroke:#00695C
```

---

## 依存関係マトリクス

| コンポーネント | 依存先 | 通信方式 | データフロー |
|--------------|--------|---------|------------|
| C8 Dashboard UI | C7 Dashboard API | REST (HTTPS) | 読み取り + 行動記録書き込み |
| C7 Dashboard API | C1 Auth | Cognito Token検証 | 認証 |
| C7 Dashboard API | C3 JudgmentEngine | Lambda呼び出し | 判定結果取得 |
| C7 Dashboard API | C4 ScoreCalculator | Lambda呼び出し | スコア取得 |
| C7 Dashboard API | C6 ActionTracker | Lambda呼び出し | 行動記録・実績取得 |
| C5 DailyReporter | C3 JudgmentEngine | Lambda呼び出し | 日次判定結果取得 |
| C5 DailyReporter | C4 ScoreCalculator | Lambda呼び出し | ダメ度スコア取得 |
| C5 DailyReporter | Slack | Webhook (HTTPS) | DM送信 |
| C5 DailyReporter | Bedrock | API呼び出し | 一言コメント生成 |
| C3 JudgmentEngine | C2 SlackConnector | DynamoDB読み取り | チャットデータ取得 |
| C3 JudgmentEngine | Bedrock | API呼び出し | 本文AI分析・判定理由生成 |
| C2 SlackConnector | Slack API | Bot Token (HTTPS) | メッセージ履歴取得 |
| C2 SlackConnector | DynamoDB | 書き込み | 正規化データ保存 |
| C6 ActionTracker | DynamoDB | 読み書き | 行動記録保存・取得 |
| C4 ScoreCalculator | DynamoDB | 読み取り | 判定結果集計 |

---

## データフロー（メインフロー）

```mermaid
sequenceDiagram
    participant SLACK as Slack API
    participant C2 as C2: SlackConnector
    participant C3 as C3: JudgmentEngine
    participant BEDROCK as Bedrock (Claude)
    participant C4 as C4: ScoreCalculator
    participant C5 as C5: DailyReporter
    participant C6 as C6: ActionTracker
    participant C7 as C7: Dashboard API
    participant C8 as C8: Dashboard UI
    participant USER as ユーザー (Slack DM)

    Note over C2,C5: 1. データ取得フロー（17:00 バッチ）
    SLACK->>C2: メッセージ履歴取得
    C2->>C2: 正規化・保存

    Note over C3,BEDROCK: 2. 判定フロー（17:30 バッチ）
    C2-->>C3: データ取得完了イベント
    C3->>C3: Layer 1: ルールベース判定
    C3->>BEDROCK: Layer 2: 本文AI分析
    BEDROCK->>C3: 判定結果 + 理由
    C3->>C3: 判定結果保存

    Note over C4: 3. スコア算出フロー（17:45）
    C3-->>C4: 判定完了イベント
    C4->>C4: ダメ度スコア算出・保存

    Note over C5,USER: 4. 定時レスポンスフロー（18:00）
    C5->>C3: 判定結果取得
    C5->>C4: スコア取得
    C5->>C6: 追跡中の行動取得
    C5->>USER: Slack DM「今日のマジサボレポート」

    Note over C8,C7: 5. ダッシュボード表示フロー（オンデマンド）
    C8->>C7: API リクエスト
    C7->>C3: 判定結果取得
    C7->>C4: スコア取得
    C7->>C6: 行動記録取得
    C7->>C8: レスポンス

    Note over C6,C2: 6. 行動追跡フロー（24h×7日）
    C6->>C2: メンションチェック依頼
    C2->>SLACK: メンション確認
    SLACK->>C2: 結果
    C2->>C6: メンション有無
```
