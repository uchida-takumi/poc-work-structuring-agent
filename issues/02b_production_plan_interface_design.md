# [Task] 予測結果を生産計画に取り込むインタフェース設計

## 親Issue
[Issue 02: 需要予測AIと生産計画連携の高度化](02_demand_forecasting_and_production_planning.md)

## 背景
需要予測AIの出力を生産計画システムへ自動・半自動で反映するためのインタフェースが存在しない。手動でのデータ転記が計画改定リードタイムの長期化を招いており、週次計画改定の半自動化を実現するAPI/データ連携設計が必要。

## 目的
- 予測結果の生産計画システムへの自動反映により、計画改定リードタイムを30%以上短縮
- 計画担当者が予測値を確認・承認後に生産計画へ反映できる半自動ワークフローを実装

## スコープ

### システム間連携アーキテクチャ

```
[需要予測AIエンジン]
        │ 週次予測バッチ出力
        ▼
[予測結果ステージング DB]
        │
        ├─ 自動反映（閾値内変動）→ [生産計画システム]
        │
        └─ 要承認フロー（閾値超変動）→ [計画担当者ダッシュボード(Issue 02d)]
                                            │ 承認/却下/修正
                                            ▼
                                        [生産計画システム]
```

### データモデル定義

#### 予測結果出力スキーマ
| フィールド | 型 | 説明 |
|-----------|---|------|
| forecast_id | UUID | 予測実行ID |
| run_timestamp | TIMESTAMP | 予測実行日時 |
| vehicle_code | STRING | 車種コード |
| region_code | STRING | 販売地域コード |
| dealer_code | STRING | 販社コード（省略可） |
| forecast_week | DATE | 予測対象週（月曜日） |
| forecast_horizon_weeks | INT | 予測ホライズン（1〜13） |
| predicted_units | DECIMAL | 予測需要台数 |
| lower_bound_80 | DECIMAL | 予測区間下限（信頼度80%） |
| upper_bound_80 | DECIMAL | 予測区間上限（信頼度80%） |
| model_version | STRING | 使用モデルバージョン |
| scenario_type | ENUM | NORMAL / PARTS_CONSTRAINT / LOGISTICS_DELAY |
| status | ENUM | PENDING / APPROVED / REJECTED / APPLIED |

#### 生産計画システムへの反映スキーマ
| フィールド | 型 | 説明 |
|-----------|---|------|
| plan_revision_id | UUID | 計画改定ID |
| source_forecast_id | UUID | 参照予測実行ID |
| vehicle_code | STRING | 車種コード |
| production_plant | STRING | 生産工場コード |
| target_week | DATE | 生産対象週 |
| planned_units | INT | 計画生産台数 |
| revision_type | ENUM | AUTO / MANUAL_APPROVED |
| applied_by | STRING | 適用者ID（自動の場合"SYSTEM"） |
| applied_at | TIMESTAMP | 適用日時 |

### APIインタフェース仕様

#### 1. 予測結果取得API
```
GET /api/v1/forecasts
  ?vehicle_code={code}
  &region_code={code}
  &forecast_week_from={date}
  &forecast_week_to={date}
  &scenario_type={NORMAL|PARTS_CONSTRAINT|LOGISTICS_DELAY}
  &status={PENDING|APPROVED|APPLIED}

Response: { forecasts: [ForecastResult], total: INT, page: INT }
```

#### 2. 予測承認API（計画担当者操作）
```
POST /api/v1/forecasts/{forecast_id}/approve
Body: { adjusted_units: DECIMAL (optional), comment: STRING }

Response: { forecast_id, status: "APPROVED", plan_revision_id }
```

#### 3. 生産計画反映トリガーAPI
```
POST /api/v1/plan-revisions
Body: { forecast_ids: [UUID], apply_mode: "AUTO"|"MANUAL" }

Response: { plan_revision_id, applied_count: INT, skipped_count: INT }
```

### 自動反映ルール
| 条件 | アクション |
|------|-----------|
| 予測変動 ≤ 前週比±5% かつ MAPE閾値内 | 自動承認・自動反映 |
| 予測変動 > 前週比±5% または 需要急変フラグON | 計画担当者へ要承認通知 |
| 部品制約シナリオ or 物流遅延シナリオ | 常に手動承認必須 |
| モデル信頼度スコア低下（異常検知） | 自動反映を一時停止・アラート発報 |

### エラーハンドリング・監査ログ
- 連携失敗時のリトライポリシー（最大3回、指数バックオフ）
- すべての反映操作の監査ログ保持（誰が・いつ・何を承認/拒否したか）
- 生産計画システム側の受入バリデーション結果をコールバックで記録

### セキュリティ要件
- APIアクセスはOAuth 2.0 / JWT認証
- 承認操作は計画担当者ロール以上に限定
- 自動反映ジョブはサービスアカウントによる認証

## 完了条件（Definition of Done）
- インタフェース仕様書（API定義、データスキーマ、連携フロー図）が承認済み
- 自動反映ルールが計画部門・IT部門の合意を得ている
- ステージング環境でのE2E連携テストが完了
- 監査ログ設計がセキュリティ・コンプライアンス部門の承認済み
- エラーハンドリングとリトライ仕様が確定

## 依存関係
- Issue 02a（需要予測モデル設計）: 予測出力スキーマの確定
- Issue 02c（シナリオ比較）: シナリオタイプの定義
- Issue 02d（ダッシュボード）: 承認ワークフローUI
- 生産計画システムの担当IT部門: 受入APIの共同設計

## リスク

| リスク | 影響度 | 対策 |
|--------|--------|------|
| 生産計画システムのAPI非対応 | 高 | CSVバッチ連携をフォールバックとして並行設計 |
| 承認フロー遅延による計画反映遅れ | 中 | エスカレーションタイムアウト設定（24h未承認で上長へ自動通知） |
| データ型不整合によるバリデーションエラー | 低〜中 | 入出力スキーマの契約テスト（Consumer-Driven Contract Testing） |

## 推奨ラベル
`task` `planning` `interface` `api` `medium-priority`
