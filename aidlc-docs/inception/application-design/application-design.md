# Application Design（統合ドキュメント）

> **サービス名**: マジメニサボル（マジサボ）
> **アーキテクチャ**: マイクロサービス（8 Unit × 独立CDKスタック）
> **フロントエンド**: React SPA（CloudFront + S3）
> **最終更新**: 2026-05-06

---

## アーキテクチャ概要

```mermaid
flowchart TB
    subgraph User["👤 ユーザー"]
        Browser[ブラウザ]
        SlackApp[Slack]
    end

    subgraph Frontend["U8: Dashboard UI"]
        CF[CloudFront + S3<br/>React SPA]
    end

    subgraph APILayer["U7: Dashboard API"]
        APIGW[API Gateway<br/>+ Cognito Authorizer]
    end

    subgraph Auth["U1: Auth"]
        COGNITO[Cognito<br/>User Pool]
    end

    subgraph SlackConn["U2: Slack Connector"]
        SC_LAMBDA[Lambda<br/>データ取得/パース/メンションチェック]
        SC_DB[(DynamoDB<br/>SlackData<br/>ChannelConfig)]
        SC_S3[S3<br/>エクスポートファイル]
    end

    subgraph Judgment["U3: Judgment Engine"]
        JE_LAMBDA[Lambda<br/>簡易判定/詳細判定]
        JE_DB[(DynamoDB<br/>Judgments)]
        BEDROCK[Amazon Bedrock<br/>Claude]
    end

    subgraph Score["U4: Score Calculator"]
        SCORE_LAMBDA[Lambda<br/>スコア算出]
        SCORE_DB[(DynamoDB<br/>Scores<br/>SaboriLevels)]
    end

    subgraph Reporter["U5: Daily Reporter"]
        DR_LAMBDA[Lambda<br/>レポート生成/リマインド]
        DR_DB[(DynamoDB<br/>ReportHistory<br/>CommentPool)]
    end

    subgraph Action["U6: Action Tracker"]
        AT_LAMBDA[Lambda<br/>行動記録/メンションチェック]
        AT_DB[(DynamoDB<br/>Actions<br/>MentionChecks)]
    end

    subgraph Infra["共通インフラ"]
        EB[EventBridge<br/>スケジューラー]
    end

    subgraph External["外部サービス"]
        SLACK_API[Slack API<br/>Bot Token]
        SLACK_WH[Slack Webhook<br/>DM送信]
    end

    %% ユーザーフロー
    Browser --> CF
    CF --> APIGW
    APIGW --> COGNITO
    SlackApp -.->|18:00 DM受信| User

    %% API → 各Unit
    APIGW --> JE_LAMBDA
    APIGW --> SCORE_LAMBDA
    APIGW --> AT_LAMBDA
    APIGW --> SC_LAMBDA

    %% Slack Connector
    SC_LAMBDA --> SLACK_API
    SC_LAMBDA --> SC_DB
    SC_LAMBDA --> SC_S3

    %% Judgment Engine
    JE_LAMBDA --> BEDROCK
    JE_LAMBDA --> JE_DB

    %% Score Calculator
    SCORE_LAMBDA --> SCORE_DB

    %% Daily Reporter
    DR_LAMBDA --> SLACK_WH
    DR_LAMBDA --> DR_DB

    %% Action Tracker
    AT_LAMBDA --> AT_DB

    %% EventBridge スケジュール
    EB -->|17:00 バッチ取得| SC_LAMBDA
    EB -->|17:30 バッチ判定| JE_LAMBDA
    EB -->|17:45 スコア算出| SCORE_LAMBDA
    EB -->|18:00 レポート配信| DR_LAMBDA
    EB -->|24h メンションチェック| AT_LAMBDA

    %% Unit間通信（EventBridge）
    SC_LAMBDA -.->|データ取得完了イベント| JE_LAMBDA
    JE_LAMBDA -.->|判定完了イベント| SCORE_LAMBDA
    AT_LAMBDA -.->|メンションチェック依頼| SC_LAMBDA

    %% スタイル
    style User fill:#E3F2FD,stroke:#1565C0
    style Frontend fill:#E8F5E9,stroke:#2E7D32
    style APILayer fill:#F3E5F5,stroke:#6A1B9A
    style Auth fill:#FFF3E0,stroke:#E65100
    style SlackConn fill:#E0F7FA,stroke:#00695C
    style Judgment fill:#FCE4EC,stroke:#880E4F
    style Score fill:#F1F8E9,stroke:#33691E
    style Reporter fill:#FFF8E1,stroke:#F57F17
    style Action fill:#EDE7F6,stroke:#4527A0
    style Infra fill:#ECEFF1,stroke:#37474F
    style External fill:#FBE9E7,stroke:#BF360C
```

---

## コンポーネント構成（8コンポーネント = 8 Unit）

| Unit | コンポーネント | 責務 | 技術 |
|:---:|--------------|------|------|
| U1 | Auth | 認証・セッション管理 | Cognito |
| U2 | SlackConnector | Slackデータ取得・正規化・チャンネル選択UI・メンションチェック | Lambda + Slack API |
| U3 | JudgmentEngine | 3分類判定＋理由生成（ハイブリッド: ルールベース+AI） | Lambda + Bedrock |
| U4 | ScoreCalculator | ダメ度スコア算出・サボりレベル管理 | Lambda |
| U5 | DailyReporter | 定時レスポンス生成・配信・リマインド・チャレンジ提示 | EventBridge + Lambda |
| U6 | ActionTracker | 行動記録・メンション追跡（24h×7日）・結果判定 | Lambda + EventBridge |
| U7 | Dashboard API | REST API提供（BFF） | API Gateway + Lambda |
| U8 | Dashboard UI | ダッシュボード＋サボりロードマップ | React (S3 + CloudFront) |

詳細: [components.md](./components.md)

---

## サービス構成（6サービス）

| # | サービス | トリガー | 主要処理 |
|---|---------|---------|---------|
| S1 | DataIngestion | Events API + EventBridge(17:00) | Slackデータ取得・正規化 |
| S2 | Judgment | S1完了 + EventBridge(17:30) | 3分類判定（ルールベース簡易+AI詳細） |
| S3 | Score | S2完了(17:45) | ダメ度スコア算出・レベル判定 |
| S4 | Report | EventBridge(18:00) | 定時レスポンス＋チャレンジ生成・配信 |
| S5 | Action | ユーザー操作 + EventBridge(24h×7日) | 行動記録・メンション追跡 |
| S6 | Dashboard | ユーザーリクエスト | データ提供API |

詳細: [services.md](./services.md)

---

## データ保持方針（マイクロサービス原則）

| 方針 | 内容 |
|------|------|
| **各Unitが自分のデータを保持** | Unit間でDynamoDBテーブルを直接参照しない |
| **データが必要な場合** | API経由 or EventBridgeイベントで取得 |
| **共有可能** | マスタデータ、設定値など読み取り専用データのみ |

### Unit別DynamoDBテーブル

| Unit | テーブル | 用途 |
|:---:|---------|------|
| U1 | — | Cognito User Pool（マネージド） |
| U2 | SlackData, ChannelConfig | 正規化チャットデータ、チャンネル設定 |
| U3 | Judgments | 判定結果（分類、理由、確信度） |
| U4 | Scores, SaboriLevels | 日次スコア、サボりレベル・ロードマップ進捗 |
| U5 | ReportHistory, CommentPool | レポート配信履歴、コメントプール |
| U6 | Actions, MentionChecks | 行動記録、メンションチェック結果 |
| U7 | — | データ保持なし（BFF、他Unitに委譲） |
| U8 | — | データ保持なし（フロントエンド） |

---

## 判定エンジンの動作モード（技術的工夫）

### ハイブリッド判定アーキテクチャ

単純な「Bedrock丸投げ」ではなく、**ルールベース + AI の2層構造**：

```
Layer 1: ルールベース判定（高速・低コスト）
  ├─ 時間外発言 → 偽勤勉（確定）
  ├─ メンション0回チャンネル → 神怠惰（確定）
  ├─ 自分宛メンション + 質問キーワード → 罪怠惰（確定）
  └─ 上記に該当しない → Layer 2へ

Layer 2: AI判定（Bedrock Claude）
  ├─ 本文の意図分析（質問/依頼/報告/雑談の分類）
  ├─ コンテキスト判定（報告系は反応なしでも正常）
  ├─ 判定理由の自然言語生成
  └─ 確信度スコア付与
```

**技術的工夫のポイント**:
1. Layer 1で70%のケースを処理 → Bedrockコスト削減
2. Layer 2は曖昧なケースのみ → AI判定の精度が高い領域に集中
3. 判定理由を必ず生成 → ユーザーの学習に直結
4. 確信度スコア → 低確信度は「判定保留」にして誤判定を防止

---

## コアフロー優先方針

### 予選MVP実装の優先順位

8 Unitを全て完璧に作るのではなく、**コアフロー（データの流れ）を最優先**で動かす。

```
【最優先】コアフロー（これが動けばデモ成立）:
  U2 Slack取得 → U3 判定 → U4 スコア算出 → U5 レポート配信

【次優先】ユーザー体験:
  U1 認証 → U8 ダッシュボード → U7 API

【最後】追跡機能:
  U6 行動追跡（メンションチェック）
```

### フォールバック方針

| 項目 | 目標 | フォールバック（5/22時点で未達の場合） |
|------|------|------|
| フロントエンド | React SPA | HTML + JavaScript + Chart.js |
| Slack連携 | Slack App（Bot Token） | エクスポートファイルアップロードのみ |
| AI判定 | Bedrock Claude | ルールベースのみ（Layer 1だけで動かす） |
| 定時レスポンス | EventBridge + Slack DM | 手動実行でデモ |

**判断日: 5月22日** — この時点でReactが動いていなければHTML+JSに切り替え。

---

## 設計上の決定事項

| 決定 | 理由 |
|------|------|
| React SPA（5/22までに判断） | ダッシュボード+ロードマップUIに最適。経験なしだがKiroで生成 |
| Slack App（Bot Token）メイン | リアルタイムデータ取得可能。エクスポートはフォールバック |
| ハイブリッド判定（ルール+AI） | コスト最適化＋技術的深さのアピール |
| Slack DM配信 | ユーザーが普段使っているSlackに届く |
| シードデータ + 実Slackデータ | デモ用にシードデータ＋実データでも動くことを見せる |
| 本文は分析後に破棄 | プライバシー配慮 |
| Unit単位データ保持 | マイクロサービス原則。直接テーブル参照禁止 |
| コアフロー優先 | U2→U3→U4→U5が動けばデモ成立 |
| 初学者モード | データなしでも一般Tipsを配信。初日から価値提供 |
| デモ用コメントプール | 一言コメントは事前用意。品質担保 |

---

## 依存関係

詳細: [component-dependency.md](./component-dependency.md)

## Unit分解

詳細: [unit-of-work.md](./unit-of-work.md)、[unit-of-work-dependency.md](./unit-of-work-dependency.md)、[unit-of-work-story-map.md](./unit-of-work-story-map.md)
