# マスターデータ統合ルール仕様書

## 概要

本書は、サプライチェーン統合データ基盤において扱う4種類のマスターデータ（部品・拠点・サプライヤー・車種）の統合ルールを定義する。

---

## 1. マスターデータ統合の基本方針

### 1.1 ゴールデンレコード方式

複数ソースから収集されたマスターデータは「ゴールデンレコード（Golden Record）」を生成する。各ソースシステムのマスターはサービスオーナーシステムとして優先順位を設定し、競合フィールドは優先ルールに従って解決する。

```
優先順位決定フロー:
  1. 各フィールドごとに「マスター権威ソース（System of Record）」を指定
  2. 権威ソース不在 or データ欠損の場合→ 次点ソースを使用
  3. 全ソース欠損の場合→ 空値（NULL）を記録し品質アラートを発報
```

### 1.2 マスターコード体系

統合データ基盤では、各ソースシステムのローカルコードに加え、**統合コード（グローバルキー）**を付与する。

| マスター種別 | 統合コード形式 | 例 |
|---|---|---|
| 部品 | `PART-{8桁数字}` | `PART-00123456` |
| 拠点 | `SITE-{ISO国コード2桁}-{4桁数字}` | `SITE-JP-0001` |
| サプライヤー | `SUPP-{10桁英数字}` | `SUPP-JP00012345` |
| 車種 | `VEH-{5桁英数字}` | `VEH-AB001` |

### 1.3 マスターデータ変更管理

- マスターの**新規登録・変更・廃止**はソースシステムで行い、統合基盤は受動的に同期する
- 廃止レコードは物理削除しない。`is_active = false` + `valid_to` 日付を設定
- 変更履歴はすべて監査ログとして保持（保管期間：7年）

---

## 2. 部品マスター統合ルール

### 2.1 ソースシステムと優先順位

| フィールド | 権威ソース | 補完ソース |
|---|---|---|
| 品目コード（ローカル） | SAP ERP | — |
| 品目名称（日本語） | SAP ERP | MES |
| 品目名称（英語） | SAP ERP | — |
| 品目区分（原材料/半完成品/製品） | SAP ERP | MES |
| 標準単価 | SAP ERP | — |
| 調達リードタイム（日） | SAP ERP | TMS実績 |
| 安全在庫数量 | SAP ERP | WMS |
| 重量（kg） | CAD/設計システム | SAP ERP |
| 梱包単位 | WMS | SAP ERP |
| 危険物区分 | 品質システム | SAP ERP |

### 2.2 統合データモデル（parts_master）

```sql
CREATE TABLE curated.parts_master (
    part_global_id      VARCHAR(12)     NOT NULL,   -- 統合コード
    part_erp_code       VARCHAR(18)     NOT NULL,   -- SAP品目コード
    part_mes_code       VARCHAR(20),                -- MES内部コード
    part_wms_code       VARCHAR(20),                -- WMS内部コード
    part_name_ja        NVARCHAR(100)   NOT NULL,
    part_name_en        VARCHAR(100),
    part_category       VARCHAR(20)     NOT NULL,   -- RAW / WIP / FG
    standard_unit_price DECIMAL(15,2),
    currency            CHAR(3)         DEFAULT 'JPY',
    procurement_lead_days INT,
    safety_stock_qty    DECIMAL(15,3),
    weight_kg           DECIMAL(10,3),
    packing_unit        INT,
    hazmat_class        VARCHAR(10),
    is_active           BOOLEAN         NOT NULL    DEFAULT TRUE,
    valid_from          DATE            NOT NULL,
    valid_to            DATE,
    created_at          TIMESTAMP       NOT NULL,
    updated_at          TIMESTAMP       NOT NULL,
    source_system       VARCHAR(20)     NOT NULL,
    PRIMARY KEY (part_global_id)
);
```

### 2.3 名寄せ（マッチング）ルール

1. **完全一致:** SAP品目コードで一意に特定（最優先）
2. **代替コードマッチング:** 旧コード・図番コードによる突合
3. **名称マッチング:** 品目名称の正規化後テキスト類似度（Levenshtein距離 ≦ 0.1）
4. **未マッチ:** 人手確認キューに登録（品質アラートカテゴリ：`MASTER_UNMATCHED`）

---

## 3. 拠点マスター統合ルール

### 3.1 ソースシステムと優先順位

| フィールド | 権威ソース | 補完ソース |
|---|---|---|
| 拠点コード（ローカル） | SAP ERP | — |
| 拠点名（日本語） | SAP ERP | WMS |
| 拠点名（英語） | SAP ERP | — |
| 拠点区分 | SAP ERP | WMS |
| 住所（国/都市/郵便番号） | SAP ERP | TMS |
| GPS座標 | GISシステム | TMS |
| キャパシティ（㎡/台数） | WMS | — |
| 親拠点（グループ拠点） | SAP ERP | — |
| タイムゾーン | GISシステム | SAP ERP |

### 3.2 統合データモデル（sites_master）

```sql
CREATE TABLE curated.sites_master (
    site_global_id      VARCHAR(12)     NOT NULL,   -- 統合コード: SITE-JP-0001
    site_erp_code       VARCHAR(10)     NOT NULL,   -- SAP拠点コード
    site_wms_code       VARCHAR(20),
    site_tms_code       VARCHAR(20),
    site_name_ja        NVARCHAR(100)   NOT NULL,
    site_name_en        VARCHAR(100),
    site_type           VARCHAR(20)     NOT NULL,   -- FACTORY / WAREHOUSE / DC / SUPPLIER
    country_code        CHAR(2)         NOT NULL,   -- ISO 3166-1 alpha-2
    prefecture          NVARCHAR(20),
    city                NVARCHAR(50),
    postal_code         VARCHAR(10),
    address             NVARCHAR(200),
    latitude            DECIMAL(9,6),
    longitude           DECIMAL(9,6),
    timezone            VARCHAR(50)     DEFAULT 'Asia/Tokyo',
    capacity_sqm        DECIMAL(10,2),
    parent_site_id      VARCHAR(12),
    is_active           BOOLEAN         NOT NULL    DEFAULT TRUE,
    valid_from          DATE            NOT NULL,
    valid_to            DATE,
    created_at          TIMESTAMP       NOT NULL,
    updated_at          TIMESTAMP       NOT NULL,
    PRIMARY KEY (site_global_id)
);
```

### 3.3 拠点階層ルール

```
拠点階層（例）:
  本社（SITE-JP-0001）
  ├─ 第1工場（SITE-JP-0010）
  │   ├─ 部品倉庫A（SITE-JP-0011）
  │   └─ 完成品倉庫B（SITE-JP-0012）
  └─ 第2工場（SITE-JP-0020）
```

- 階層は最大5レベルまでサポート
- ループ参照は許可しない（制約チェックあり）

---

## 4. サプライヤーマスター統合ルール

### 4.1 ソースシステムと優先順位

| フィールド | 権威ソース | 補完ソース |
|---|---|---|
| サプライヤーコード | SAP ERP | — |
| サプライヤー名（正式） | SAP ERP | — |
| サプライヤー区分 | 調達管理システム | SAP ERP |
| 与信状態 | 財務システム | 調達管理システム |
| 納入実績評価スコア | 調達管理システム | 品質システム |
| 品質評価スコア | 品質システム | — |
| 主担当連絡先 | 調達管理システム | — |
| 取扱品目 | SAP ERP | — |
| 認証・資格（ISO9001等） | 品質システム | — |

### 4.2 統合データモデル（suppliers_master）

```sql
CREATE TABLE curated.suppliers_master (
    supplier_global_id      VARCHAR(14)     NOT NULL,   -- 統合コード: SUPP-JP00012345
    supplier_erp_code       VARCHAR(10)     NOT NULL,   -- SAP仕入先コード
    supplier_qms_code       VARCHAR(20),
    supplier_name           NVARCHAR(100)   NOT NULL,
    supplier_name_short     NVARCHAR(50),
    supplier_type           VARCHAR(20)     NOT NULL,   -- TIER1 / TIER2 / TIER3
    country_code            CHAR(2)         NOT NULL,
    credit_status           VARCHAR(20),                -- NORMAL / WATCH / SUSPENDED
    delivery_score          DECIMAL(5,2),               -- 0.0-100.0
    quality_score           DECIMAL(5,2),               -- 0.0-100.0
    overall_risk_score      DECIMAL(5,2),               -- 算出値（加重平均）
    certifications          JSONB,                      -- ["ISO9001","IATF16949"]
    primary_contact_email   VARCHAR(200),
    is_active               BOOLEAN         NOT NULL    DEFAULT TRUE,
    valid_from              DATE            NOT NULL,
    valid_to                DATE,
    created_at              TIMESTAMP       NOT NULL,
    updated_at              TIMESTAMP       NOT NULL,
    PRIMARY KEY (supplier_global_id)
);
```

### 4.3 重複排除ルール

1. **法人番号マッチング:** 日本の法人番号（13桁）で同一法人を特定
2. **名称正規化マッチング:** 株式会社→(株) 等の表記統一後に比較
3. **住所マッチング:** 正規化住所 + 電話番号で補完確認
4. **統合後の対応関係管理:** 旧コードと統合コードのマッピングテーブルを保持

---

## 5. 車種マスター統合ルール

### 5.1 ソースシステムと優先順位

| フィールド | 権威ソース | 補完ソース |
|---|---|---|
| 車種コード | 製品企画システム | SAP ERP |
| 車種名称 | 製品企画システム | — |
| 車種区分（セダン/SUV等） | 製品企画システム | — |
| 生産拠点 | SAP ERP | MES |
| 生産開始/終了日 | 製品企画システム | MES |
| 標準BOM（親） | SAP ERP | — |
| 販売市場（国/地域） | 販売管理システム | — |
| 排気量/動力方式 | 製品企画システム | — |

### 5.2 統合データモデル（vehicle_models_master）

```sql
CREATE TABLE curated.vehicle_models_master (
    vehicle_global_id       VARCHAR(10)     NOT NULL,   -- 統合コード: VEH-AB001
    vehicle_erp_code        VARCHAR(18)     NOT NULL,
    vehicle_planning_code   VARCHAR(20),
    model_name_ja           NVARCHAR(50)    NOT NULL,
    model_name_en           VARCHAR(50),
    model_category          VARCHAR(20),                -- SEDAN/SUV/TRUCK/EV
    powertrain_type         VARCHAR(20),                -- ICE/HEV/PHEV/BEV/FCEV
    engine_displacement_cc  INT,
    production_site_ids     JSONB,                      -- ["SITE-JP-0010","SITE-JP-0020"]
    production_start_date   DATE,
    production_end_date     DATE,
    sales_markets           JSONB,                      -- ["JP","US","EU"]
    bom_id                  VARCHAR(20),                -- 標準BOMリファレンス
    is_active               BOOLEAN         NOT NULL    DEFAULT TRUE,
    created_at              TIMESTAMP       NOT NULL,
    updated_at              TIMESTAMP       NOT NULL,
    PRIMARY KEY (vehicle_global_id)
);
```

---

## 6. マスターデータ品質ルール

| ルール | 対象マスター | チェック方法 | 許容閾値 |
|---|---|---|---|
| 必須フィールド非NULL | 全マスター | スキーマバリデーション | 0件 |
| 統合コード重複なし | 全マスター | UNIQUE制約 | 0件 |
| 親子参照整合性 | 拠点（親拠点） | 外部キー制約 | 0件 |
| 有効日付ロジック（valid_from ≦ valid_to） | 全マスター | 日付比較 | 0件 |
| 未マッチレコード率 | 部品/サプライヤー | 名寄せ処理後の未確認率 | ≦ 1% |
| マスタ非存在トランザクション率 | 全トランザクション | 結合チェック | ≦ 0.5% |

---

## 承認履歴

| バージョン | 日付 | 作成者 | 承認者 | 変更内容 |
|---|---|---|---|---|
| 1.0 | 2026-03-12 | データ基盤チーム | IT部門長・各マスタオーナー | 初版作成 |
