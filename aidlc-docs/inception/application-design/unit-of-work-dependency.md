# Unit of Work 依存関係マトリクス

---

## 依存関係マトリクス

| Unit | U1 Auth | U2 Slack | U3 Judgment | U4 Score | U5 Reporter | U6 Action | U7 API | U8 UI |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **U1 Auth** | — | | | | | | | |
| **U2 Slack** | ○ | — | | | | | | |
| **U3 Judgment** | | ● | — | | | | | |
| **U4 Score** | | | ● | — | | | | |
| **U5 Reporter** | | | ○ | ○ | — | ○ | | |
| **U6 Action** | | ○ | | | | — | | |
| **U7 API** | ● | ○ | ○ | ○ | | ○ | — | |
| **U8 UI** | ○ | | | | | | ● | — |

**凡例**: ● = 強い依存（必須）、○ = 弱い依存（データ参照）

---

## 依存関係図

```
U8 Dashboard UI
    │
    │ REST API
    ▼
U7 Dashboard API ──── U1 Auth（トークン検証）
    │
    ├──→ U3 Judgment（判定結果取得）
    ├──→ U4 Score（スコア取得）
    ├──→ U6 Action（行動記録）
    └──→ U2 Slack（設定情報）

U5 Daily Reporter
    │
    ├──→ U3 Judgment（日次判定結果）
    ├──→ U4 Score（ダメ度スコア）
    ├──→ U6 Action（追跡中の行動）
    └──→ Slack Webhook（DM送信）

U3 Judgment Engine
    │
    ├──→ U2 Slack Connector（チャットデータ）
    └──→ Bedrock（AI分析）

U4 Score Calculator
    │
    └──→ U3 Judgment（判定結果集計）

U6 Action Tracker
    │
    └──→ U2 Slack Connector（メンションチェック）
```

---

## Unit間通信方式

| 通信パターン | 方式 | 使用箇所 |
|------------|------|---------|
| **同期呼び出し** | Lambda直接呼び出し or API Gateway | U7→U3/U4/U6（ダッシュボード表示時） |
| **非同期イベント** | EventBridge | U2→U3（データ取得完了→判定開始）、U3→U4（判定完了→スコア算出） |
| **スケジュール** | EventBridge Schedule | U5（18:00レポート）、U6（24hメンションチェック） |
| **外部Webhook** | HTTPS | U5→Slack DM |

### データ保持方針（マイクロサービス原則）

| 方針 | 内容 |
|------|------|
| **各Unitが自分のデータを保持** | Unit間でDynamoDBテーブルを直接参照しない |
| **データが必要な場合** | API経由 or EventBridgeイベントで取得 |
| **共有可能** | マスタデータ、設定値など読み取り専用データのみ |
| **例: U3がチャットデータを必要とする場合** | U2のAPIを呼び出す or U2がEventBridgeでデータを発行 → U3が受信して自Unit内に保存 |

---

## デプロイ順序

Unit間の依存関係から、デプロイ順序は以下の通り：

```
Phase 1（依存なし）:
  U1 Auth（Cognito）

Phase 2（U1に依存）:
  U2 Slack Connector
  U8 Dashboard UI（認証SDK参照のみ）

Phase 3（U2に依存）:
  U3 Judgment Engine
  U6 Action Tracker

Phase 4（U3に依存）:
  U4 Score Calculator

Phase 5（U3/U4/U6に依存）:
  U5 Daily Reporter
  U7 Dashboard API
```

---

## 結合ポイント（設計確定）

| 結合ポイント | 方式 | 備考 |
|------------|------|------|
| U2→U3 データ連携 | EventBridge（データ取得完了イベント発行） | U3はイベント受信後、U2のAPIでデータ取得 |
| U3→U4 判定完了通知 | EventBridge（判定完了イベント発行） | U4はイベント受信後、U3のAPIで判定結果取得 |
| U7→各Unit データ取得 | 各UnitのAPI呼び出し（Lambda Invoke） | BFFパターン。U7が各Unitを集約 |
| U5→Slack DM送信 | Slack Webhook（HTTPS） | エラー時はリトライ（3回） |
| U6→U2 メンションチェック | U2のAPI呼び出し | U6がU2にメンション有無を問い合わせ |
