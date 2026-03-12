# 共通データ辞書

## 概要

本書は、サプライチェーン統合データ基盤で使用する共通用語・コード体系・フィールド定義を一元管理するデータ辞書である。ドメイン横断での用語統一と、AIモデル開発・分析作業の標準化を目的とする。

---

## 1. ドメイン共通フィールド定義

### 1.1 全テーブル共通フィールド

| フィールド名 | データ型 | 説明 | 入力例 |
|---|---|---|---|
| `source_system` | VARCHAR(20) | データ発生元システム識別子 | `SAP_ERP`, `MES_MAIN`, `WMS_ORACLE`, `TMS_SAP`, `QMS_SIEMENS` |
| `created_at` | TIMESTAMP(UTC) | レコード作成日時 | `2026-03-12T00:00:00Z` |
| `updated_at` | TIMESTAMP(UTC) | レコード最終更新日時 | `2026-03-12T06:30:00Z` |
| `quality_flag` | VARCHAR(20) | データ品質フラグ | `NORMAL`, `WARNING`, `ERROR`, `MANUAL_EXCLUDE` |
| `quality_check_code` | VARCHAR(50) | 品質チェックルールコード | `DQ_ACC_INV_NEGATIVE_QTY` |
| `is_imputed` | BOOLEAN | 欠損補完実施フラグ | `false` |

### 1.2 マスタ共通フィールド

| フィールド名 | データ型 | 説明 | 入力例 |
|---|---|---|---|
| `is_active` | BOOLEAN | 有効/廃止フラグ | `true` |
| `valid_from` | DATE | 有効開始日 | `2026-01-01` |
| `valid_to` | DATE | 有効終了日（NULLは無期限） | `2026-12-31` |

---

## 2. コード体系定義

### 2.1 ソースシステムコード

| コード | システム名 | 対応ドメイン |
|---|---|---|
| `SAP_ERP` | SAP ERP（調達・財務モジュール） | 調達、マスタ |
| `MES_MAIN` | 製造実行システム（メイン工場） | 生産 |
| `MES_SUB` | 製造実行システム（サブ工場） | 生産 |
| `WMS_ORACLE` | Oracle Warehouse Management | 在庫 |
| `TMS_SAP` | SAP Transportation Management | 物流 |
| `QMS_SIEMENS` | Siemens Opcenter Quality | 品質 |
| `EXT_WEATHER` | 外部気象API | 物流（外部） |
| `EXT_PORT` | 外部港湾情報API | 物流（外部） |
| `MANUAL` | 手動入力 | 全ドメイン |

### 2.2 品目区分コード（part_category）

| コード | 名称（日本語） | 名称（英語） | 説明 |
|---|---|---|---|
| `RAW` | 原材料 | Raw Material | 加工前の素材 |
| `WIP` | 仕掛品 | Work In Progress | 製造途中の部品 |
| `COMP` | 購買部品 | Purchased Component | 外部調達の完成部品 |
| `SUB_ASSY` | サブアセンブリ | Sub-Assembly | 中間組立品 |
| `FG` | 完成品 | Finished Goods | 出荷可能な製品 |
| `SPARE` | サービスパーツ | Spare Parts | 保守用部品 |
| `CONSUMABLE` | 消耗品 | Consumable | 製造補助消耗品 |

### 2.3 拠点区分コード（site_type）

| コード | 名称（日本語） | 名称（英語） | 説明 |
|---|---|---|---|
| `FACTORY` | 工場 | Factory | 製造拠点 |
| `WAREHOUSE` | 倉庫 | Warehouse | 在庫保管拠点 |
| `DC` | 物流センター | Distribution Center | 仕分け・配送拠点 |
| `SUPPLIER` | サプライヤー拠点 | Supplier Site | 仕入先の製造・発送拠点 |
| `PORT` | 港湾 | Port | 海上・航空輸送拠点 |
| `DEALER` | 販売店 | Dealer | 顧客納品先（参照用） |

### 2.4 サプライヤー区分コード（supplier_type）

| コード | 名称（日本語） | 名称（英語） | 説明 |
|---|---|---|---|
| `TIER1` | 一次サプライヤー | Tier 1 Supplier | 直接調達先 |
| `TIER2` | 二次サプライヤー | Tier 2 Supplier | Tier1の調達先 |
| `TIER3` | 三次サプライヤー | Tier 3 Supplier | Tier2の調達先 |
| `LOGISTICS` | 物流会社 | Logistics Provider | 輸送・倉庫業者 |
| `SERVICE` | サービス業者 | Service Provider | 設備保守等のサービス業者 |

### 2.5 輸送手段コード（transport_mode）

| コード | 名称（日本語） | 名称（英語） | 標準リードタイム目安 |
|---|---|---|---|
| `TRUCK` | トラック | Truck | 国内: 1-3日 |
| `AIR` | 航空 | Air Freight | 国際: 1-3日 |
| `SEA` | 海上 | Sea Freight | 国際: 15-45日 |
| `RAIL` | 鉄道 | Rail | 国内: 2-5日 |
| `COURIER` | 宅配便 | Courier | 国内: 翌日 |

### 2.6 品質検査区分コード（inspection_type）

| コード | 名称（日本語） | 名称（英語） | タイミング |
|---|---|---|---|
| `INCOMING` | 受入検査 | Incoming Inspection | 入庫時 |
| `IN_PROCESS` | 工程内検査 | In-Process Inspection | 工程中 |
| `FINAL` | 最終検査 | Final Inspection | 出荷前 |
| `AUDIT` | 監査検査 | Audit Inspection | 定期/抜き取り |

### 2.7 不良区分コード（defect_category）

| コード | 名称（日本語） | 名称（英語） | 説明 |
|---|---|---|---|
| `FUNCTIONAL` | 機能不良 | Functional Defect | 動作・性能に影響 |
| `APPEARANCE` | 外観不良 | Appearance Defect | 外観・仕上げの不具合 |
| `DIMENSIONAL` | 寸法不良 | Dimensional Defect | 寸法・公差外れ |
| `MATERIAL` | 材質不良 | Material Defect | 材料・成分の不具合 |
| `ASSEMBLY` | 組付不良 | Assembly Defect | 組立・取付の不具合 |

### 2.8 在庫移動区分コード（movement_type）

| コード | 名称（日本語） | 名称（英語） | 説明 |
|---|---|---|---|
| `GR` | 入庫 | Goods Receipt | 外部からの入庫 |
| `GI` | 出庫 | Goods Issue | 生産・出荷への払出 |
| `TRANSFER` | 移送 | Stock Transfer | 拠点間・倉庫間移動 |
| `ADJUSTMENT` | 在庫調整 | Inventory Adjustment | 棚卸差異の調整 |
| `RETURN` | 返品 | Return | 仕入先・顧客への返品 |
| `SCRAP` | 廃棄 | Scrap | 不良品・期限切れ廃棄 |

### 2.9 出荷ステータスコード（shipment_status）

| コード | 名称（日本語） | 名称（英語） | 説明 |
|---|---|---|---|
| `PLANNED` | 計画済み | Planned | 出荷指示発行済み |
| `DEPARTED` | 出発済み | Departed | 発送完了 |
| `IN_TRANSIT` | 輸送中 | In Transit | 輸送中 |
| `ARRIVED` | 到着済み | Arrived | 受領完了 |
| `DELAYED` | 遅延 | Delayed | 遅延中（SLA超過） |
| `CANCELLED` | キャンセル | Cancelled | 出荷キャンセル |

### 2.10 製造指示ステータスコード（wo_status）

| コード | 名称（日本語） | 名称（英語） | 説明 |
|---|---|---|---|
| `PLANNED` | 計画済み | Planned | 製造指示発行前 |
| `RELEASED` | 解放済み | Released | 製造指示発行済み |
| `IN_PROCESS` | 生産中 | In Process | 製造着手済み |
| `COMPLETED` | 完了 | Completed | 製造完了 |
| `CANCELLED` | キャンセル | Cancelled | 中止 |
| `PARTIAL` | 一部完了 | Partially Completed | 部分完了（残数あり） |

---

## 3. 計量単位コード（unit_of_measure）

| コード | 名称（日本語） | SI単位 | 用途 |
|---|---|---|---|
| `PC` | 個 | — | 部品・完成品 |
| `KG` | キログラム | kg | 重量管理品 |
| `M` | メートル | m | 線材・シート |
| `M2` | 平方メートル | m² | 板材・シート |
| `M3` | 立方メートル | m³ | 液体・気体 |
| `L` | リットル | L | 液体（少量） |
| `SET` | セット | — | セット品 |
| `BOX` | 箱 | — | 包装単位 |
| `PALLET` | パレット | — | 物流単位 |

---

## 4. 通貨コード（ISO 4217準拠）

| コード | 名称 | 使用シーン |
|---|---|---|
| `JPY` | 日本円 | 国内調達・生産 |
| `USD` | 米ドル | 北米調達・輸出 |
| `EUR` | ユーロ | 欧州調達 |
| `CNY` | 人民元 | 中国調達 |

---

## 5. AIユースケース向けKPI定義

### 5.1 需要予測KPI

| KPI名 | 定義 | 計算式 | 目標値 |
|---|---|---|---|
| MAPE（平均絶対誤差率） | 予測精度 | Σ\|実績-予測\|/実績 / n × 100 | ≦ 15% |
| Bias | 系統的な予測の偏り | Σ(予測-実績) / n | ≒ 0 |
| 予測カバレッジ | 予測値が存在するSKU率 | 予測済みSKU / 全対象SKU × 100 | ≧ 95% |

### 5.2 在庫最適化KPI

| KPI名 | 定義 | 計算式 | 目標値 |
|---|---|---|---|
| 欠品率 | 欠品が発生した日の割合 | 欠品日数 / 総営業日数 × 100 | ≦ 2% |
| 過剰在庫比率 | 安全在庫を大幅超過した在庫の割合 | 過剰在庫金額 / 総在庫金額 × 100 | ≦ 15% |
| 在庫回転率 | 在庫の回転速度 | 出庫量 / 平均在庫量 | ≧ 12回/年 |
| 在庫日数（DOS） | 現在庫で何日分の需要を賄えるか | 在庫数量 / 日平均需要量 | 品目別最適値 |

### 5.3 物流最適化KPI

| KPI名 | 定義 | 計算式 | 目標値 |
|---|---|---|---|
| OTIF率 | 期日通り・数量通りの納品率 | OTIF件数 / 総出荷件数 × 100 | ≧ 95% |
| 遅延率 | 納期遅延の割合 | 遅延件数 / 総出荷件数 × 100 | ≦ 5% |
| 緊急輸送比率 | 航空便等の緊急輸送の割合 | 緊急輸送費 / 総輸送費 × 100 | ≦ 3% |
| 平均遅延時間（遅延便のみ） | 遅延発生時の平均遅延時間 | Σ遅延時間 / 遅延件数 | ≦ 24時間 |

### 5.4 品質KPI

| KPI名 | 定義 | 計算式 | 目標値 |
|---|---|---|---|
| 不良率（PPM） | 百万個当たりの不良数 | 不良数 / 検査数 × 1,000,000 | ≦ 500 PPM |
| 合格率 | 検査合格の割合 | 合格数 / 検査数 × 100 | ≧ 99.95% |
| 根本原因特定時間 | 不良発生から原因特定までの時間 | 平均時間（時） | ≦ 48時間 |
| 予兆検知リードタイム | 問題発生前の検知リードタイム（AI） | 平均リードタイム（時） | ≧ 4時間 |

### 5.5 サプライヤーリスクKPI

| KPI名 | 定義 | 計算式 | 目標値 |
|---|---|---|---|
| 納入遅延率 | 約束納期を超過した入庫の割合 | 遅延入庫件数 / 総入庫件数 × 100 | ≦ 5% |
| 品質不合格率 | 受入検査不合格の割合 | 不合格件数 / 受入件数 × 100 | ≦ 1% |
| 総合サプライヤースコア | 納期・品質・コストの総合評価 | 加重平均スコア（0-100） | ≧ 80 |

---

## 6. 用語集（Glossary）

| 用語（日本語） | 用語（英語） | 定義 |
|---|---|---|
| 共通データ基盤 | Common Data Platform | 複数ドメインのデータを統合・管理する基盤インフラ |
| ゴールデンレコード | Golden Record | 複数ソースから統合された、信頼できる唯一の真実のデータ |
| CDC | Change Data Capture | データベースの変更（INSERT/UPDATE/DELETE）をリアルタイムに捕捉する技術 |
| データレイク | Data Lake | 生データから加工済みデータまでを一元保管するストレージ |
| 特徴量ストア | Feature Store | AIモデル学習・推論に使用する特徴量を一元管理するシステム |
| ERP | Enterprise Resource Planning | 調達・生産・財務等を統合管理する基幹業務システム |
| MES | Manufacturing Execution System | 製造現場の工程・実績を管理する製造実行システム |
| WMS | Warehouse Management System | 倉庫内の在庫・作業を管理する倉庫管理システム |
| TMS | Transportation Management System | 輸送計画・実績を管理する輸送管理システム |
| QMS | Quality Management System | 品質基準・検査・改善を管理する品質管理システム |
| BOM | Bill of Materials | 完成品を構成する部品の一覧と所要数量を示した部品表 |
| SPC | Statistical Process Control | 統計的手法を用いて製造工程を監視・制御する品質管理手法 |
| OTIF | On-Time In-Full | 期日通りかつ数量通りに納品が完了した割合を示すKPI |
| MAPE | Mean Absolute Percentage Error | 予測精度を測る指標。予測誤差の絶対値の平均値（%） |
| DoS | Days of Supply | 現在の在庫数量で何日分の需要を賄えるかを示す指標 |
| PPM | Parts Per Million | 百万個あたりの不良個数を示す品質指標 |
| Safety Stock | 安全在庫 | 需要・供給のばらつきによる欠品を防ぐための緩衝在庫量 |
| Reorder Point | 発注点 | この在庫水準になったら発注を行う基準値 |
| Lead Time | リードタイム | 発注から納入完了（または製造完了）までの所要時間 |

---

## 承認履歴

| バージョン | 日付 | 作成者 | 承認者 | 変更内容 |
|---|---|---|---|---|
| 1.0 | 2026-03-12 | データ基盤チーム | IT部門長・各業務部門長 | 初版作成 |
