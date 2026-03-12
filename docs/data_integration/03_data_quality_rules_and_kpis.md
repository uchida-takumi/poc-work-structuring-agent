# データ品質ルールおよびKPI定義書

## 概要

本書は、サプライチェーン統合データ基盤におけるデータ品質の管理ルール、チェック項目、監視指標（KPI）を定義する。データ品質の4類型（欠損・重複・遅延・異常値）それぞれについて具体的な基準と対処フローを規定する。

---

## 1. データ品質の基本方針

### 1.1 データ品質次元

| 品質次元 | 定義 | 主要測定指標 |
|---|---|---|
| **完全性（Completeness）** | 必須データが欠損なく存在すること | 欠損率 |
| **一意性（Uniqueness）** | 重複レコードが存在しないこと | 重複率 |
| **適時性（Timeliness）** | 定義されたSLA内にデータが到達すること | SLA遵守率、平均遅延時間 |
| **正確性（Accuracy）** | 値が業務的に正しい範囲・形式であること | 異常値検出率 |
| **一貫性（Consistency）** | 複数システム間でデータが矛盾しないこと | マスタ不整合率 |
| **有効性（Validity）** | 定義されたデータ型・フォーマットに従っていること | スキーマ違反率 |

### 1.2 品質スコアリング

総合データ品質スコアは以下の加重平均で算出する：

```
品質スコア = 完全性(25%) + 一意性(15%) + 適時性(25%) + 正確性(20%) + 一貫性(10%) + 有効性(5%)

目標: ≧ 90点（AIモデル学習に使用可能な基準）
警告: 80-89点（使用可能だが改善要）
不合格: < 80点（AIモデルへの投入停止）
```

---

## 2. 欠損（Completeness）ルール

### 2.1 欠損チェック定義

| データ系統 | テーブル/フィールド | 欠損判定条件 | 許容欠損率 | 対処アクション |
|---|---|---|---|---|
| 調達 | purchase_orders.delivery_date | NULL または過去3年以上前の日付 | ≦ 0.5% | アラート発報→ERP担当確認 |
| 調達 | goods_receipt.quantity | NULL または 0 | ≦ 0.1% | レコード除外＋アラート |
| 生産 | work_orders.planned_quantity | NULL | ≦ 0.2% | アラート発報→MES担当確認 |
| 生産 | production_results.actual_quantity | NULL | ≦ 1.0% | 代替値補完（計画値×効率係数） |
| 在庫 | stock_snapshots.quantity | NULL | ≦ 0.1% | 前日値引継ぎ＋アラート |
| 物流 | shipment_tracking.actual_arrival | NULLかつ予定到着から72時間超過 | ≦ 2.0% | エスカレーションアラート |
| 品質 | inspection_results.result_code | NULL | ≦ 0.5% | 保留ステータスに設定＋アラート |

### 2.2 欠損補完ポリシー

欠損補完は原則として**行わない**（AIモデルが扱うため、欠損を明示的に保持する）。ただし、運用上必要な場合は以下の例外的補完を適用し、必ず `is_imputed = true` フラグを付与する。

| 補完方法 | 適用条件 | 対象フィールド |
|---|---|---|
| 前値引継ぎ（Forward Fill） | 時系列で前値が存在する場合 | 在庫数量、設備状態 |
| 計画値代入 | 実績欠損で計画値が存在する場合 | 生産数量（暫定） |
| グループ平均代入 | 同品目・同拠点の直近30日平均 | 標準サイクルタイム |

---

## 3. 重複（Uniqueness）ルール

### 3.1 重複チェック定義

| データ系統 | テーブル | 重複キー定義 | 許容重複率 | 対処アクション |
|---|---|---|---|---|
| 調達 | purchase_orders | (po_number, line_item) | 0件 | 最新タイムスタンプを採用、旧レコードを論理削除 |
| 調達 | goods_receipt | (gr_document_number, gr_line) | 0件 | 重複除外＋アラート発報 |
| 生産 | work_orders | (wo_number, plant_code) | 0件 | 最新レコード採用 |
| 在庫 | stock_snapshots | (part_code, site_code, snapshot_date) | 0件 | 同日複数スナップショット→最終スナップを採用 |
| 物流 | shipments | (shipment_number) | 0件 | 重複除外＋アラート |
| 品質 | inspection_results | (inspection_number, inspection_seq) | 0件 | 重複除外＋アラート |
| マスタ | parts_master | (part_global_id) | 0件 | 名寄せキューに登録 |

### 3.2 重複検出アルゴリズム

```python
# 重複検出の基本ロジック（Spark）
from pyspark.sql import functions as F
from pyspark.sql.window import Window

def detect_duplicates(df, key_columns, timestamp_col):
    """
    重複を検出し、最新レコードを保持する
    """
    window = Window.partitionBy(key_columns).orderBy(F.desc(timestamp_col))
    df_ranked = df.withColumn("row_rank", F.row_number().over(window))
    
    duplicates = df_ranked.filter(F.col("row_rank") > 1)
    deduplicated = df_ranked.filter(F.col("row_rank") == 1).drop("row_rank")
    
    return deduplicated, duplicates
```

---

## 4. 遅延（Timeliness）ルール

### 4.1 SLA遅延チェック定義

| データ系統 | データ種別 | SLA目標 | 遅延警告閾値 | 遅延エラー閾値 |
|---|---|---|---|---|
| 調達 | 入庫実績（CDC） | 5分以内 | 10分超過 | 30分超過 |
| 調達 | 日次バッチ（PO/マスタ） | 07:00 JST | 07:30 JST | 08:00 JST |
| 生産 | WOイベント（CDC） | 2分以内 | 5分超過 | 15分超過 |
| 生産 | 設備テレメトリ | 30秒以内 | 1分超過 | 5分超過 |
| 在庫 | 在庫移動（CDC） | 3分以内 | 5分超過 | 15分超過 |
| 在庫 | 日次スナップショット | 06:00 JST | 06:30 JST | 07:00 JST |
| 物流 | 輸送状態更新 | 5分以内 | 10分超過 | 30分超過 |
| 品質 | 検査実績（CDC） | 3分以内 | 5分超過 | 15分超過 |

### 4.2 遅延計測方法

```
データ遅延 = データ取込完了時刻 - ソースシステムでの事象発生時刻

計測ポイント:
  1. ソースシステムでの事象発生タイムスタンプ（source_event_time）
  2. Kafkaトピックへのメッセージ投入時刻（kafka_ingestion_time）
  3. Data Lakeへの書込完了時刻（lake_write_time）
  4. 標準化レイヤーへの反映完了時刻（standardized_time）
```

### 4.3 遅延対応フロー

```
遅延検知
  │
  ├─ 警告（Warning）: PagerDutyへ通知 → データ基盤チームが調査開始（30分以内）
  │
  └─ エラー（Error）: PagerDutyへ緊急通知 → 関係システムチームへ連絡
                       → SLA違反としてインシデントレポートに記録
                       → バックフィル処理のトリガー
```

---

## 5. 異常値（Accuracy）ルール

### 5.1 統計的異常値検出

| データ系統 | フィールド | 検出方法 | 閾値 | 対処アクション |
|---|---|---|---|---|
| 在庫 | 在庫数量 | Z-Score（同品目・拠点の直近90日） | \|z\| > 3.0 | フラグ付与＋確認依頼 |
| 生産 | サイクルタイム | IQR法（同工程・同品目） | Q1-3×IQR以下 または Q3+3×IQR以上 | フラグ付与＋MES確認 |
| 物流 | 輸送リードタイム | 正規分布（同レーン直近180日） | μ±3σ外 | フラグ付与 |
| 品質 | 検査測定値（連続値） | SPC管理限界（UCL/LCL） | 管理限界外 | 即時アラート＋品質担当通知 |
| 調達 | 発注単価 | 移動平均（直近24ヶ月） | ±30%超変動 | 確認依頼フラグ |

### 5.2 業務ルールベース異常値検出

```sql
-- 業務ルール異常値チェック例
-- ルール1: 在庫数量は非負
SELECT * FROM stock_snapshots WHERE quantity < 0;

-- ルール2: 入庫日は発注日以降
SELECT gr.* FROM goods_receipt gr
JOIN purchase_orders po ON gr.po_number = po.po_number
WHERE gr.actual_receipt_date < po.order_date;

-- ルール3: 出荷数量は在庫数量を超えない
SELECT s.* FROM shipments s
JOIN stock_snapshots ss ON s.part_code = ss.part_code AND s.from_site = ss.site_code
WHERE s.shipped_quantity > ss.quantity * 1.1;  -- 10%バッファを許容

-- ルール4: 検査合格率が100%超え
SELECT * FROM inspection_results
WHERE passed_qty > inspected_qty;
```

### 5.3 異常値フラグ仕様

異常値はレコードを削除せず、以下のフラグを付与して保持する：

```sql
-- 異常値フラグカラム（全テーブル共通）
quality_flag        VARCHAR(20)     -- NORMAL / WARNING / ERROR / MANUAL_EXCLUDE
quality_check_code  VARCHAR(50)     -- 検出ルールコード（例: DQ_ACC_INV_NEGATIVE_QTY）
quality_checked_at  TIMESTAMP       -- チェック実施時刻
is_imputed          BOOLEAN         -- 補完済みフラグ
```

---

## 6. データ品質KPI

### 6.1 KPI一覧

| KPI名 | 計算式 | 目標値 | 計測頻度 |
|---|---|---|---|
| **欠損率** | NULL件数 / 全件数 × 100 | ≦ 1.0% | 日次 |
| **重複率** | 重複レコード件数 / 全件数 × 100 | ≦ 0.01% | 日次 |
| **SLA遵守率（リアルタイム）** | SLA内到達件数 / 全件数 × 100 | ≧ 95% | 時次 |
| **SLA遵守率（バッチ）** | SLA遵守日数 / 全稼働日数 × 100 | ≧ 99% | 日次 |
| **マスタ不整合率** | マスタ不在トランザクション件数 / 全トランザクション × 100 | ≦ 0.5% | 日次 |
| **異常値発生率** | 異常フラグ付レコード件数 / 全件数 × 100 | ≦ 2.0% | 日次 |
| **スキーマ違反率** | スキーマ違反件数 / 全件数 × 100 | ≦ 0.01% | リアルタイム |
| **総合データ品質スコア** | 加重平均スコア（1.2節参照） | ≧ 90点 | 日次 |

### 6.2 KPIダッシュボード仕様

```
ダッシュボード構成（Power BI）:

1. サマリービュー
   - 総合品質スコア（ゲージ）
   - 主要KPI一覧（トレンドグラフ 直近30日）
   - アクティブアラート一覧

2. 系統別ビュー（調達/生産/在庫/物流/品質）
   - 欠損率ヒートマップ（テーブル × フィールド）
   - SLA遵守率（時系列）
   - 異常値発生件数（棒グラフ）

3. マスタ品質ビュー
   - マスタ不整合率（系統別）
   - 未名寄せレコード件数
   - マスタ変更履歴

4. インシデントビュー
   - 未解決品質インシデント一覧
   - SLA違反履歴
   - エスカレーション状況
```

### 6.3 品質レポーティングサイクル

| レポート種別 | 頻度 | 宛先 | 内容 |
|---|---|---|---|
| 日次品質サマリー | 毎営業日 09:00 | データ基盤チーム | KPI全体・前日比・アラート状況 |
| 週次品質レポート | 毎週月曜 10:00 | IT部門長・業務システムオーナー | KPIトレンド・品質インシデント・改善状況 |
| 月次品質レポート | 毎月第1営業日 | 経営層・プロジェクトスポンサー | 月次KPI達成状況・リスク報告 |

---

## 7. 品質アラート対応フロー

```
品質アラート発報
  │
  ├─ 重大度: CRITICAL（KPI閾値を大幅超過、AIモデル影響あり）
  │     └─ 即時対応（1時間以内）→ データ基盤チームリーダー + 該当システムオーナー
  │
  ├─ 重大度: HIGH（KPI閾値超過、業務影響あり）
  │     └─ 当日対応（4時間以内）→ データ基盤チーム + 該当システム担当
  │
  └─ 重大度: MEDIUM / LOW（軽微な品質低下）
        └─ 翌営業日対応 → データ基盤チームの定例作業として処理
```

---

## 承認履歴

| バージョン | 日付 | 作成者 | 承認者 | 変更内容 |
|---|---|---|---|---|
| 1.0 | 2026-03-12 | データ基盤チーム | IT部門長・品質管理部門 | 初版作成 |
