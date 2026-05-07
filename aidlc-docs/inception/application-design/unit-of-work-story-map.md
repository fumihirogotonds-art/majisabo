# Unit of Work × ストーリーマッピング

---

## ストーリー → Unit マッピング

| ストーリー | 内容 | 担当Unit | スコープ |
|-----------|------|---------|---------|
| US-01 | 認証 | **U1** auth-unit | 予選Must |
| US-02 | チャット連携 | **U2** slack-connector-unit | 予選Must |
| US-03 | チャット判定 | **U3** judgment-engine-unit | 予選Must |
| US-04 | 行動記録 | **U6** action-tracker-unit | 予選Must |
| US-05 | ダッシュボード＋ダメ度 | **U7** + **U8** + **U4** | 予選Must |
| US-06 | 定時レスポンス | **U5** daily-reporter-unit | 予選Must |
| US-07 | メール連携 | **U2** slack-connector-unit（拡張） | 予選Should |
| US-08 | メール判定 | **U3** judgment-engine-unit（拡張） | 予選Should |
| US-09 | アクション評価 | **U3** + **U7** + **U8** | 予選Should |
| US-10 | チーム招待 | **U1**（拡張）+ **U7** | 決勝 |
| US-11 | チームダッシュボード | **U4**（拡張）+ **U7** + **U8** | 決勝 |
| US-12 | チーム改善提案 | **U3**（拡張）+ **U5** | 決勝 |
| US-13 | 怠惰ノウハウ共有 | **U5**（拡張） | 決勝 |
| US-14 | 組織KPIレポート | **U4**（拡張）+ **U7** | 決勝 |

---

## Unit → ストーリー マッピング

| Unit | 予選Must | 予選Should | 決勝 |
|------|---------|-----------|------|
| **U1** auth-unit | US-01 | | US-10 |
| **U2** slack-connector-unit | US-02 | US-07 | |
| **U3** judgment-engine-unit | US-03 | US-08, US-09 | US-12 |
| **U4** score-calculator-unit | US-05（一部） | | US-11, US-14 |
| **U5** daily-reporter-unit | US-06 | | US-12, US-13 |
| **U6** action-tracker-unit | US-04 | | |
| **U7** dashboard-api-unit | US-05（一部） | US-09 | US-10, US-11, US-14 |
| **U8** dashboard-ui-unit | US-05（一部） | US-09 | US-11 |

---

## 予選MVP実装順序（推奨）

Unit間の依存関係とストーリーの優先度から、以下の順序で実装：

```
Week 1（基盤）:
  ├─ U1 auth-unit（US-01）         ← 全Unitの前提
  ├─ U2 slack-connector-unit（US-02）← データ取得が全ての起点
  └─ U8 dashboard-ui-unit（US-05の画面部分）← 並行してUI構築開始

Week 2（コア機能）:
  ├─ U3 judgment-engine-unit（US-03）← U2のデータを使って判定
  ├─ U4 score-calculator-unit（US-05のスコア部分）← U3の結果を集計
  ├─ U6 action-tracker-unit（US-04）← 行動記録＋メンションチェック
  └─ U7 dashboard-api-unit（US-05のAPI部分）← フロントとバックを繋ぐ

Week 2.5（仕上げ）:
  ├─ U5 daily-reporter-unit（US-06）← 全Unitのデータを使ってレポート生成
  └─ 結合テスト＋デモデータ投入
```

---

## チーム分担案（3人）

| メンバー | 担当Unit | スキル要件 |
|---------|---------|-----------|
| **メンバーA（インフラ）** | U1, U2, U5, U6 | CDK、Lambda、EventBridge、Slack API |
| **メンバーB（バックエンド）** | U3, U4, U7 | Lambda、Bedrock、DynamoDB、API Gateway |
| **メンバーC（フロントエンド）** | U8 | React、Vite、Chart.js |

---

## 審査基準「Unit分解の適切さ」へのアピールポイント

1. **分割理由が明確**: 各Unitが単一責務（SRP）を持ち、独立してデプロイ可能
2. **依存関係が一方向**: データフローが U2→U3→U4→U5 と一方向。循環依存なし
3. **スケーラビリティ**: 各Unitが独立してスケール可能（判定エンジンだけ負荷が高ければそこだけスケール）
4. **チーム並行開発**: 3人が異なるUnitを並行して開発可能
5. **段階的拡張**: 予選→決勝で各Unitを「拡張」するだけ。新Unitの追加は不要
6. **テスト容易性**: Unit単位でモックを使ったテストが可能
