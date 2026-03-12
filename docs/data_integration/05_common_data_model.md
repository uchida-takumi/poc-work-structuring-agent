# 共通データモデル設計書

## 概要

本書は、サプライチェーン統合データ基盤における分析・AI利用向けの共通データモデルを定義する。Raw（生データ）→ Standardized（標準化）→ Curated（キュレーション）の3層アーキテクチャを採用し、各AIユースケースが共通的に参照できるデータモデルを設計する。

---

## 1. データレイヤーアーキテクチャ

```
┌────────────────────────────────────────────────────────────────────┐
│  Layer 1: Raw（生データ層）                                         │
│  - ソースシステムのデータをそのまま保管（スキーマ変更なし）          │
│  - パーティション: system_code / data_type / yyyy / mm / dd         │
│  - フォーマット: Avro（CDC）、CSV（バッチ）、JSON（API）            │
│  - 保管期間: 7年（コンプライアンス要件）                            │
└─────────────────────────────┬──────────────────────────────────────┘
                              │ ETL（型変換・文字コード正規化・NULL統一）
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│  Layer 2: Standardized（標準化層）                                  │
│  - 共通スキーマに変換・統合コードでマスタ結合済み                   │
│  - フォーマット: Parquet（Delta Lake形式）                          │
│  - パーティション: domain / entity / yyyy / mm / dd                 │
│  - 保管期間: 3年                                                    │
└─────────────────────────────┬──────────────────────────────────────┘
                              │ dbt変換（集計・特徴量生成・Joinモデル）
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│  Layer 3: Curated（キュレーション層）                               │
│  - AIモデル学習・推論・BI分析向けに最適化されたデータ               │
│  - フォーマット: Parquet / Delta Lake                               │
│  - 保管期間: 2年（ローリング）                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Analytical Mart        │  Feature Store（AI特徴量ストア）  │  │
│  │  - KPIアグリゲーション  │  - 需要予測特徴量                 │  │
│  │  - 時系列サマリー       │  - 在庫最適化特徴量               │  │
│  │  - クロスドメイン結合   │  - 遅延予測特徴量                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

---

## 2. 標準化層（Standardized）データモデル

### 2.1 調達ドメインモデル

```sql
-- 購買発注ヘッダー
CREATE TABLE standardized.purchase_orders (
    po_global_id            VARCHAR(20)     NOT NULL,
    po_erp_number           VARCHAR(10)     NOT NULL,
    vendor_global_id        VARCHAR(14)     NOT NULL,   -- SUPP-xxxx
    purchasing_org          VARCHAR(10)     NOT NULL,
    purchasing_group        VARCHAR(3),
    po_date                 DATE            NOT NULL,
    currency                CHAR(3)         NOT NULL,
    total_amount            DECIMAL(17,2),
    po_status               VARCHAR(20)     NOT NULL,   -- OPEN / CLOSED / CANCELLED
    created_at              TIMESTAMP       NOT NULL,
    updated_at              TIMESTAMP       NOT NULL,
    source_system           VARCHAR(20)     NOT NULL,
    quality_flag            VARCHAR(20)     DEFAULT 'NORMAL',
    PRIMARY KEY (po_global_id)
);

-- 購買発注明細
CREATE TABLE standardized.purchase_order_items (
    po_item_global_id       VARCHAR(24)     NOT NULL,
    po_global_id            VARCHAR(20)     NOT NULL,
    po_item_number          INT             NOT NULL,
    part_global_id          VARCHAR(12)     NOT NULL,   -- PART-xxxx
    site_global_id          VARCHAR(12)     NOT NULL,   -- SITE-JP-xxxx
    ordered_quantity        DECIMAL(15,3)   NOT NULL,
    unit_of_measure         VARCHAR(3)      NOT NULL,
    unit_price              DECIMAL(15,5),
    scheduled_delivery_date DATE            NOT NULL,
    actual_delivery_date    DATE,
    open_quantity           DECIMAL(15,3),
    po_item_status          VARCHAR(20)     NOT NULL,
    source_system           VARCHAR(20)     NOT NULL,
    quality_flag            VARCHAR(20)     DEFAULT 'NORMAL',
    PRIMARY KEY (po_item_global_id)
);

-- 入庫実績
CREATE TABLE standardized.goods_receipts (
    gr_global_id            VARCHAR(24)     NOT NULL,
    gr_document_number      VARCHAR(10)     NOT NULL,
    gr_line_number          INT             NOT NULL,
    po_item_global_id       VARCHAR(24),
    part_global_id          VARCHAR(12)     NOT NULL,
    site_global_id          VARCHAR(12)     NOT NULL,
    received_quantity       DECIMAL(15,3)   NOT NULL,
    unit_of_measure         VARCHAR(3)      NOT NULL,
    receipt_timestamp       TIMESTAMP       NOT NULL,
    receipt_type            VARCHAR(20)     NOT NULL,   -- STANDARD / RETURN / REJECTION
    source_system           VARCHAR(20)     NOT NULL,
    quality_flag            VARCHAR(20)     DEFAULT 'NORMAL',
    PRIMARY KEY (gr_global_id)
);
```

### 2.2 生産ドメインモデル

```sql
-- 製造指示
CREATE TABLE standardized.work_orders (
    wo_global_id            VARCHAR(20)     NOT NULL,
    wo_erp_number           VARCHAR(12)     NOT NULL,
    vehicle_global_id       VARCHAR(10),                -- VEH-xxxx
    part_global_id          VARCHAR(12)     NOT NULL,
    site_global_id          VARCHAR(12)     NOT NULL,
    process_code            VARCHAR(20)     NOT NULL,
    planned_start           TIMESTAMP       NOT NULL,
    planned_end             TIMESTAMP       NOT NULL,
    planned_quantity        DECIMAL(15,3)   NOT NULL,
    actual_start            TIMESTAMP,
    actual_end              TIMESTAMP,
    actual_good_quantity    DECIMAL(15,3),
    actual_defect_quantity  DECIMAL(15,3),
    wo_status               VARCHAR(20)     NOT NULL,   -- PLANNED / IN_PROCESS / COMPLETED / CANCELLED
    source_system           VARCHAR(20)     NOT NULL,
    quality_flag            VARCHAR(20)     DEFAULT 'NORMAL',
    PRIMARY KEY (wo_global_id)
);

-- 設備テレメトリ（時系列）
CREATE TABLE standardized.equipment_telemetry (
    equipment_id            VARCHAR(20)     NOT NULL,
    site_global_id          VARCHAR(12)     NOT NULL,
    process_code            VARCHAR(20),
    measurement_timestamp   TIMESTAMP       NOT NULL,
    temperature_c           DECIMAL(8,2),
    current_a               DECIMAL(8,3),
    vibration_mm_s          DECIMAL(8,3),
    rotation_rpm            DECIMAL(10,2),
    operating_status        VARCHAR(20),                -- RUNNING / IDLE / FAULT / MAINTENANCE
    quality_flag            VARCHAR(20)     DEFAULT 'NORMAL',
    PRIMARY KEY (equipment_id, measurement_timestamp)
)
PARTITION BY (measurement_timestamp);
```

### 2.3 在庫ドメインモデル

```sql
-- 在庫スナップショット（日次確定値）
CREATE TABLE standardized.inventory_snapshots (
    snapshot_date           DATE            NOT NULL,
    part_global_id          VARCHAR(12)     NOT NULL,
    site_global_id          VARCHAR(12)     NOT NULL,
    location_code           VARCHAR(20),
    on_hand_quantity        DECIMAL(15,3)   NOT NULL,
    reserved_quantity       DECIMAL(15,3)   DEFAULT 0,
    available_quantity      DECIMAL(15,3)   NOT NULL,
    unit_of_measure         VARCHAR(3)      NOT NULL,
    inventory_value_jpy     DECIMAL(17,2),
    lot_number              VARCHAR(20),
    source_system           VARCHAR(20)     NOT NULL,
    quality_flag            VARCHAR(20)     DEFAULT 'NORMAL',
    PRIMARY KEY (snapshot_date, part_global_id, site_global_id)
);

-- 在庫移動トランザクション
CREATE TABLE standardized.inventory_movements (
    movement_global_id      VARCHAR(24)     NOT NULL,
    movement_timestamp      TIMESTAMP       NOT NULL,
    part_global_id          VARCHAR(12)     NOT NULL,
    from_site_global_id     VARCHAR(12),
    to_site_global_id       VARCHAR(12),
    movement_type           VARCHAR(20)     NOT NULL,   -- GR / GI / TRANSFER / ADJUSTMENT
    quantity                DECIMAL(15,3)   NOT NULL,
    unit_of_measure         VARCHAR(3)      NOT NULL,
    reference_document      VARCHAR(20),
    source_system           VARCHAR(20)     NOT NULL,
    quality_flag            VARCHAR(20)     DEFAULT 'NORMAL',
    PRIMARY KEY (movement_global_id)
);
```

### 2.4 物流ドメインモデル

```sql
-- 出荷（Shipment）
CREATE TABLE standardized.shipments (
    shipment_global_id      VARCHAR(20)     NOT NULL,
    shipment_erp_number     VARCHAR(10)     NOT NULL,
    from_site_global_id     VARCHAR(12)     NOT NULL,
    to_site_global_id       VARCHAR(12)     NOT NULL,
    transport_mode          VARCHAR(20)     NOT NULL,   -- TRUCK / AIR / SEA / RAIL
    carrier_global_id       VARCHAR(14),
    planned_departure       TIMESTAMP       NOT NULL,
    actual_departure        TIMESTAMP,
    planned_arrival         TIMESTAMP       NOT NULL,
    actual_arrival          TIMESTAMP,
    shipment_status         VARCHAR(20)     NOT NULL,   -- PLANNED / DEPARTED / IN_TRANSIT / ARRIVED / DELAYED
    total_weight_kg         DECIMAL(10,3),
    delay_risk_score        DECIMAL(5,3),               -- AI出力: 0.0-1.0
    source_system           VARCHAR(20)     NOT NULL,
    quality_flag            VARCHAR(20)     DEFAULT 'NORMAL',
    PRIMARY KEY (shipment_global_id)
);

-- 出荷明細
CREATE TABLE standardized.shipment_items (
    shipment_item_global_id VARCHAR(24)     NOT NULL,
    shipment_global_id      VARCHAR(20)     NOT NULL,
    part_global_id          VARCHAR(12)     NOT NULL,
    shipped_quantity        DECIMAL(15,3)   NOT NULL,
    unit_of_measure         VARCHAR(3)      NOT NULL,
    lot_number              VARCHAR(20),
    source_system           VARCHAR(20)     NOT NULL,
    quality_flag            VARCHAR(20)     DEFAULT 'NORMAL',
    PRIMARY KEY (shipment_item_global_id)
);
```

### 2.5 品質ドメインモデル

```sql
-- 検査実績
CREATE TABLE standardized.inspection_results (
    inspection_global_id    VARCHAR(24)     NOT NULL,
    inspection_timestamp    TIMESTAMP       NOT NULL,
    inspection_type         VARCHAR(20)     NOT NULL,   -- INCOMING / IN_PROCESS / FINAL
    part_global_id          VARCHAR(12)     NOT NULL,
    site_global_id          VARCHAR(12)     NOT NULL,
    wo_global_id            VARCHAR(20),
    lot_number              VARCHAR(20),
    inspected_quantity      DECIMAL(15,3)   NOT NULL,
    passed_quantity         DECIMAL(15,3)   NOT NULL,
    defect_quantity         DECIMAL(15,3)   NOT NULL,
    defect_code             VARCHAR(20),
    defect_category         VARCHAR(20),                -- FUNCTIONAL / APPEARANCE / DIMENSIONAL
    inspector_id            VARCHAR(20),
    source_system           VARCHAR(20)     NOT NULL,
    quality_flag            VARCHAR(20)     DEFAULT 'NORMAL',
    PRIMARY KEY (inspection_global_id)
);

-- SPC測定データ（時系列）
CREATE TABLE standardized.spc_measurements (
    measurement_point_id    VARCHAR(20)     NOT NULL,
    measurement_timestamp   TIMESTAMP       NOT NULL,
    process_code            VARCHAR(20)     NOT NULL,
    site_global_id          VARCHAR(12)     NOT NULL,
    part_global_id          VARCHAR(12),
    measured_value          DECIMAL(15,6)   NOT NULL,
    ucl                     DECIMAL(15,6),
    lcl                     DECIMAL(15,6),
    target_value            DECIMAL(15,6),
    is_out_of_control       BOOLEAN         NOT NULL    DEFAULT FALSE,
    source_system           VARCHAR(20)     NOT NULL,
    quality_flag            VARCHAR(20)     DEFAULT 'NORMAL',
    PRIMARY KEY (measurement_point_id, measurement_timestamp)
)
PARTITION BY (measurement_timestamp);
```

---

## 3. キュレーション層（Curated）データモデル

### 3.1 需要予測特徴量テーブル

```sql
CREATE TABLE curated.demand_forecast_features (
    feature_date            DATE            NOT NULL,
    part_global_id          VARCHAR(12)     NOT NULL,
    site_global_id          VARCHAR(12)     NOT NULL,
    vehicle_global_id       VARCHAR(10),

    -- 需要実績特徴量
    actual_production_qty_d1    DECIMAL(15,3),  -- 前日実績
    actual_production_qty_d7    DECIMAL(15,3),  -- 7日前実績
    production_qty_ma7          DECIMAL(15,3),  -- 7日移動平均
    production_qty_ma28         DECIMAL(15,3),  -- 28日移動平均
    production_qty_yoy          DECIMAL(15,3),  -- 前年同期比

    -- 発注・在庫特徴量
    open_po_quantity            DECIMAL(15,3),  -- 未着発注残
    on_hand_quantity            DECIMAL(15,3),  -- 手持在庫
    days_of_supply              DECIMAL(8,2),   -- 在庫日数
    safety_stock_qty            DECIMAL(15,3),

    -- 外部特徴量
    day_of_week                 INT,
    is_holiday                  BOOLEAN,
    month                       INT,

    -- ラベル
    actual_demand_next7days     DECIMAL(15,3),  -- 目的変数（翌7日需要）
    actual_demand_next28days    DECIMAL(15,3),  -- 目的変数（翌28日需要）

    created_at                  TIMESTAMP       NOT NULL,
    PRIMARY KEY (feature_date, part_global_id, site_global_id)
);
```

### 3.2 在庫最適化特徴量テーブル

```sql
CREATE TABLE curated.inventory_optimization_features (
    feature_date            DATE            NOT NULL,
    part_global_id          VARCHAR(12)     NOT NULL,
    site_global_id          VARCHAR(12)     NOT NULL,

    -- 在庫特徴量
    on_hand_quantity            DECIMAL(15,3),
    days_of_supply              DECIMAL(8,2),
    stockout_days_last30        INT,            -- 直近30日の欠品日数
    overstock_ratio             DECIMAL(5,3),   -- 過剰在庫比率

    -- 調達リスク特徴量
    supplier_risk_score         DECIMAL(5,3),   -- サプライヤーリスクスコア（0-1）
    avg_po_lead_time_days       DECIMAL(8,2),   -- 平均調達リードタイム
    lead_time_variance          DECIMAL(8,2),   -- リードタイムばらつき

    -- 需要ばらつき特徴量
    demand_std_dev_28d          DECIMAL(15,3),  -- 需要標準偏差（28日）
    demand_cv                   DECIMAL(8,3),   -- 需要変動係数

    -- 推奨値（AI出力）
    recommended_safety_stock    DECIMAL(15,3),
    recommended_reorder_point   DECIMAL(15,3),
    recommended_order_quantity  DECIMAL(15,3),
    stockout_probability_7d     DECIMAL(5,4),   -- 7日以内欠品確率

    created_at                  TIMESTAMP       NOT NULL,
    PRIMARY KEY (feature_date, part_global_id, site_global_id)
);
```

### 3.3 物流遅延予測特徴量テーブル

```sql
CREATE TABLE curated.logistics_delay_features (
    feature_timestamp       TIMESTAMP       NOT NULL,
    shipment_global_id      VARCHAR(20)     NOT NULL,
    transport_lane_id       VARCHAR(20)     NOT NULL,

    -- 輸送特徴量
    transport_mode              VARCHAR(20),
    planned_lead_time_hours     DECIMAL(8,2),
    elapsed_time_ratio          DECIMAL(5,3),   -- 経過時間/計画リードタイム
    current_location_lat        DECIMAL(9,6),
    current_location_lon        DECIMAL(9,6),
    distance_to_destination_km  DECIMAL(10,2),

    -- 外部特徴量
    weather_severity_score      DECIMAL(5,3),   -- 0-1（悪天候度）
    port_congestion_index       DECIMAL(5,3),   -- 0-1（港湾混雑度）
    traffic_delay_minutes       DECIMAL(8,2),

    -- 実績特徴量（同レーン過去実績）
    lane_avg_delay_hours_30d    DECIMAL(8,2),
    lane_delay_rate_30d         DECIMAL(5,3),

    -- ラベル（AI出力）
    delay_probability           DECIMAL(5,4),   -- 遅延確率 0-1
    estimated_delay_hours       DECIMAL(8,2),   -- 推定遅延時間

    created_at                  TIMESTAMP       NOT NULL,
    PRIMARY KEY (feature_timestamp, shipment_global_id)
);
```

### 3.4 品質予兆特徴量テーブル

```sql
CREATE TABLE curated.quality_anomaly_features (
    feature_timestamp       TIMESTAMP       NOT NULL,
    process_code            VARCHAR(20)     NOT NULL,
    site_global_id          VARCHAR(12)     NOT NULL,
    part_global_id          VARCHAR(12),

    -- SPC特徴量
    spc_violation_count_1h      INT,            -- 直近1時間の管理限界外回数
    spc_trend_slope_1h          DECIMAL(10,6),  -- トレンド傾き
    spc_ma_deviation            DECIMAL(10,6),  -- 移動平均からの偏差

    -- 工程特徴量
    defect_rate_1h              DECIMAL(5,4),   -- 直近1時間不良率
    defect_rate_ma_24h          DECIMAL(5,4),   -- 24時間移動平均不良率
    cycle_time_deviation        DECIMAL(8,3),   -- サイクルタイム偏差

    -- 設備特徴量
    equipment_temperature_z     DECIMAL(8,3),   -- 温度Zスコア
    equipment_vibration_z       DECIMAL(8,3),   -- 振動Zスコア

    -- ラベル（AI出力）
    anomaly_score               DECIMAL(5,4),   -- 異常スコア 0-1
    is_alert                    BOOLEAN,        -- アラートフラグ
    alert_severity              VARCHAR(20),    -- LOW / MEDIUM / HIGH / CRITICAL

    created_at                  TIMESTAMP       NOT NULL,
    PRIMARY KEY (feature_timestamp, process_code, site_global_id)
);
```

---

## 4. 共通メタデータモデル

### 4.1 データリネージ管理

```sql
CREATE TABLE metadata.data_lineage (
    lineage_id              VARCHAR(36)     NOT NULL,   -- UUID
    source_table            VARCHAR(100)    NOT NULL,
    target_table            VARCHAR(100)    NOT NULL,
    transformation_type     VARCHAR(50)     NOT NULL,   -- CDC / BATCH / STREAM / dbt
    transformation_logic    TEXT,
    source_system           VARCHAR(20),
    run_id                  VARCHAR(36),
    run_timestamp           TIMESTAMP       NOT NULL,
    records_processed       BIGINT,
    records_failed          BIGINT          DEFAULT 0,
    status                  VARCHAR(20)     NOT NULL,   -- SUCCESS / PARTIAL / FAILED
    PRIMARY KEY (lineage_id)
);
```

### 4.2 データカタログエントリ

```sql
CREATE TABLE metadata.data_catalog (
    catalog_id              VARCHAR(36)     NOT NULL,   -- UUID
    schema_name             VARCHAR(50)     NOT NULL,
    table_name              VARCHAR(100)    NOT NULL,
    table_description_ja    NVARCHAR(500),
    table_description_en    VARCHAR(500),
    data_domain             VARCHAR(20)     NOT NULL,   -- PROCUREMENT / PRODUCTION / INVENTORY / LOGISTICS / QUALITY / MASTER
    data_sensitivity        VARCHAR(20)     NOT NULL,   -- PUBLIC / INTERNAL / CONFIDENTIAL / RESTRICTED
    pii_contains            BOOLEAN         NOT NULL    DEFAULT FALSE,
    owner_team              VARCHAR(50),
    update_frequency        VARCHAR(20),
    retention_days          INT,
    is_active               BOOLEAN         NOT NULL    DEFAULT TRUE,
    created_at              TIMESTAMP       NOT NULL,
    updated_at              TIMESTAMP       NOT NULL,
    PRIMARY KEY (catalog_id),
    UNIQUE (schema_name, table_name)
);
```

---

## 5. データモデル設計原則

| 原則 | 内容 |
|---|---|
| **冪等性** | 同じデータを複数回処理しても結果が変わらないよう設計 |
| **不変性（Immutability）** | Raw層のデータは変更しない。変換結果は別テーブルに保持 |
| **スキーマ後方互換性** | フィールド追加はOK、削除・型変更は非推奨（移行プロセス必須） |
| **統合コード一貫性** | 全テーブルでグローバルID（PART-xxx等）を外部キーとして使用 |
| **タイムゾーン統一** | 全タイムスタンプはUTCで保管し、表示時にJSTに変換 |
| **品質フラグ必須** | 全トランザクションテーブルに `quality_flag` カラムを付与 |
| **削除禁止** | レコードの物理削除は禁止。`is_active = false` または `valid_to` で廃止を表現 |

---

## 承認履歴

| バージョン | 日付 | 作成者 | 承認者 | 変更内容 |
|---|---|---|---|---|
| 1.0 | 2026-03-12 | データ基盤チーム | IT部門長・AI開発チーム | 初版作成 |
