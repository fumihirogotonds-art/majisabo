# Unit of Work 定義

> **アーキテクチャ**: マイクロサービス（8 Unit）
> **デプロイ単位**: Unit単位で別CDKスタック
> **通信方式**: EventBridge（イベント駆動）/ API呼び出し（同期）。直接テーブル参照は禁止（マイクロサービス原則）

---

## Unit一覧

| Unit | 名前 | 責務 | デプロイ単位 | 対応コンポーネント |
|:---:|------|------|:---:|:---:|
| U1 | **auth-unit** | ユーザー認証・セッション管理 | CDK Stack 1 | C1 |
| U2 | **slack-connector-unit** | Slackデータ取得・正規化・メンションチェック | CDK Stack 2 | C2 |
| U3 | **judgment-engine-unit** | 3分類判定＋判定理由生成 | CDK Stack 3 | C3 |
| U4 | **score-calculator-unit** | ダメ度スコア算出・トレンド管理 | CDK Stack 4 | C4 |
| U5 | **daily-reporter-unit** | 定時レスポンス生成・Slack DM配信 | CDK Stack 5 | C5 |
| U6 | **action-tracker-unit** | 行動記録・メンション追跡・結果判定 | CDK Stack 6 | C6 |
| U7 | **dashboard-api-unit** | REST API提供（BFF） | CDK Stack 7 | C7 |
| U8 | **dashboard-ui-unit** | React SPA（フロントエンド） | CDK Stack 8 | C8 |

---

## Unit詳細

### U1: auth-unit

| 項目 | 内容 |
|------|------|
| **責務** | ユーザー登録、ログイン、トークン発行・検証 |
| **AWSリソース** | Cognito User Pool, Cognito Identity Pool |
| **API** | なし（Cognito SDKで直接認証） |
| **データ** | Cognito User Pool（ユーザー属性にSlack連携状態を保持） |
| **依存先** | なし（他Unitから参照される） |
| **依存元** | U7（トークン検証）、U2（ユーザーID取得） |

---

### U2: slack-connector-unit

| 項目 | 内容 |
|------|------|
| **責務** | Slackワークスペースからデータ取得、正規化、保存。チャンネル選択UI用データ提供。メンションチェック |
| **AWSリソース** | Lambda × 3（データ取得、エクスポートパース、メンションチェック）、DynamoDB（ChatData, ChannelStats）、S3（エクスポートファイル保存）、EventBridge（バッチスケジュール） |
| **API** | POST /slack/connect, GET /slack/channels, POST /slack/channels/select, POST /slack/import |
| **外部連携** | Slack API（Bot Token: conversations.history, conversations.list） |
| **データ出力** | DynamoDB ChatDataテーブル、ChannelStatsテーブル |
| **依存先** | Slack API |
| **依存元** | U3（判定対象データ取得）、U6（メンションチェック） |

---

### U3: judgment-engine-unit

| 項目 | 内容 |
|------|------|
| **責務** | チャットデータを分析し、神サボリ/罪サボリ/偽マジメに分類。判定理由を生成 |
| **AWSリソース** | Lambda × 2（簡易判定、詳細判定）、DynamoDB（Judgments）、EventBridge（バッチトリガー） |
| **AI連携** | Amazon Bedrock（Claude）— 本文分析、判定理由生成 |
| **動作モード** | リアルタイム簡易（メタデータのみ）+ バッチ詳細（本文AI分析） |
| **データ入力** | U2のAPI経由でチャットデータを取得（EventBridgeイベント受信 or API呼び出し） |
| **データ出力** | DynamoDB Judgmentsテーブル（自Unit所有） |
| **依存先** | U2（チャットデータ）、Bedrock |
| **依存元** | U4（スコア算出）、U5（レポート生成）、U7（API提供） |

---

### U4: score-calculator-unit

| 項目 | 内容 |
|------|------|
| **責務** | 判定結果を集計し、ダメ度スコア算出。日次/週次トレンド管理 |
| **AWSリソース** | Lambda × 1、DynamoDB（Scores） |
| **算出式** | `(神サボリ件数 / 全判定件数) × 100 - (罪サボリ件数 × 10)` |
| **データ入力** | U3のAPI経由 or EventBridgeイベント（判定完了通知）で判定結果を取得 |
| **データ出力** | DynamoDB Scoresテーブル（自Unit所有） |
| **依存先** | U3（判定結果） |
| **依存元** | U5（レポート生成）、U7（API提供） |

---

### U5: daily-reporter-unit

| 項目 | 内容 |
|------|------|
| **責務** | 毎日18:00に日次レポート生成・Slack DM配信。48時間リマインド送信 |
| **AWSリソース** | Lambda × 2（レポート生成、リマインド送信）、EventBridge（18:00スケジュール + 48hリマインド）、DynamoDB（コメントプール） |
| **外部連携** | Slack Webhook（DM送信） |
| **AI連携** | Bedrock（本番時の一言コメント生成。デモ時はコメントプールから選択） |
| **データ入力** | U3 Judgments、U4 Scores、U6 Actions |
| **依存先** | U3、U4、U6、Slack Webhook、Bedrock |
| **依存元** | なし（最終出力） |

---

### U6: action-tracker-unit

| 項目 | 内容 |
|------|------|
| **責務** | 行動記録、メンション追跡（24h×7日）、結果判定、フィードバック |
| **AWSリソース** | Lambda × 2（行動記録、メンションチェック）、DynamoDB（Actions）、EventBridge（24hメンションチェックスケジュール） |
| **追跡ロジック** | 48hリマインド（U5経由）+ 24hメンションチェック×7日 → メンション0回で「サボり成功」確定 |
| **データ入力** | ユーザー操作（API経由）、U2（メンションチェック結果） |
| **データ出力** | DynamoDB Actionsテーブル、U3へのフィードバック |
| **依存先** | U2（メンションチェック） |
| **依存元** | U5（リマインド）、U7（API提供） |

---

### U7: dashboard-api-unit

| 項目 | 内容 |
|------|------|
| **責務** | フロントエンド向けREST API（BFF: Backend For Frontend） |
| **AWSリソース** | API Gateway、Lambda × 1（ルーティング）、Cognito Authorizer |
| **API一覧** | GET /dashboard, GET /judgments, GET /trends, GET /channels, POST /actions, POST /actions/:id/report, GET /settings, POST /settings/slack |
| **認証** | Cognito JWT Token検証 |
| **依存先** | U1（認証）、U2（Slack設定）、U3（判定結果）、U4（スコア）、U6（行動記録） |
| **依存元** | U8（フロントエンド） |

---

### U8: dashboard-ui-unit

| 項目 | 内容 |
|------|------|
| **責務** | ダッシュボード画面（React SPA） |
| **AWSリソース** | S3（静的ファイル）、CloudFront（CDN配信） |
| **画面** | ダッシュボード（ダメ度+サマリー）、Plan（チャンネル判定一覧）、履歴（トレンドグラフ）、設定（Slack連携・チャンネル選択） |
| **技術** | React + Vite + Chart.js（グラフ） |
| **依存先** | U7（API）、U1（認証SDK） |
| **依存元** | なし（最終出力） |

---

## コード構成（リポジトリ構造）

```
majisabo/
├── infrastructure/          # CDKプロジェクト
│   ├── bin/
│   │   └── app.ts          # CDKアプリエントリポイント
│   ├── lib/
│   │   ├── auth-stack.ts           # U1
│   │   ├── slack-connector-stack.ts # U2
│   │   ├── judgment-engine-stack.ts # U3
│   │   ├── score-calculator-stack.ts # U4
│   │   ├── daily-reporter-stack.ts  # U5
│   │   ├── action-tracker-stack.ts  # U6
│   │   ├── dashboard-api-stack.ts   # U7
│   │   └── dashboard-ui-stack.ts    # U8
│   └── package.json
│
├── services/                # Lambda関数（Unit別）
│   ├── slack-connector/     # U2
│   │   ├── handler.py
│   │   └── requirements.txt
│   ├── judgment-engine/     # U3
│   │   ├── handler.py
│   │   └── requirements.txt
│   ├── score-calculator/    # U4
│   │   ├── handler.py
│   │   └── requirements.txt
│   ├── daily-reporter/      # U5
│   │   ├── handler.py
│   │   └── requirements.txt
│   ├── action-tracker/      # U6
│   │   ├── handler.py
│   │   └── requirements.txt
│   └── dashboard-api/       # U7
│       ├── handler.py
│       └── requirements.txt
│
├── frontend/                # U8
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   └── App.tsx
│   ├── package.json
│   └── vite.config.ts
│
├── docs/                    # ドキュメント
│   └── ...
│
└── README.md
```


---

## コアフロー優先方針

8 Unitを全て完璧に作るのではなく、**コアフロー（データの流れ）を最優先**で動かす。

### 実装優先度

| 優先度 | Unit | 理由 |
|:---:|------|------|
| ★★★ | U2 slack-connector | 全データの起点。これがないと何も動かない |
| ★★★ | U3 judgment-engine | サービスの核心。判定がないと価値がない |
| ★★★ | U4 score-calculator | ダメ度スコアがないとダッシュボードが空 |
| ★★★ | U5 daily-reporter | デモ映え最大のポイント。18:00レポート |
| ★★☆ | U1 auth | 認証。なくてもデモは可能だが、本番には必須 |
| ★★☆ | U8 dashboard-ui | ダッシュボード。デモで見せる画面 |
| ★★☆ | U7 dashboard-api | UIとバックエンドを繋ぐ |
| ★☆☆ | U6 action-tracker | 追跡機能。コアフローが動いた後に追加 |

### フォールバック方針

| 項目 | 目標 | フォールバック（5/22時点で未達の場合） |
|------|------|------|
| フロントエンド（U8） | React SPA | HTML + JavaScript + Chart.js |
| Slack連携（U2） | Slack App（Bot Token） | エクスポートファイルアップロードのみ |
| AI判定（U3） | Bedrock Claude（ハイブリッド） | ルールベースのみ（Layer 1） |
| 定時レスポンス（U5） | EventBridge + Slack DM | 手動実行でデモ |

**判断日: 5月22日** — この時点でReactが動いていなければHTML+JSに切り替え。
