# Databricks 利用料ダッシュボード 仕様書

**文書作成日**: 2026年1月28日  
**対象ファイル**: `dash-hhhd-dbks-mng-usage_updated260115.lvdash.json`

---

## 1. 概要

本ダッシュボードは、Databricksの利用料を可視化・管理するためのダッシュボードです。ワークスペース別、請求元製品別、SKU別の利用料を期間ごとに集計し、予算管理と実績管理を行います。

### 1.1 ダッシュボード名
- **ページ名**: 利用料ダッシュボード

### 1.2 主な機能
1. 期間ごとの実績値の可視化（日次/週次/月次）
2. 累積利用料のグラフ表示（予算・閾値との比較）
3. 予算対実績の管理テーブル
4. ワークスペース別・請求元製品別・SKU別の詳細分析

---

## 2. データソース（テーブル）

### 2.1 使用テーブル一覧

| スキーマ | テーブル名 | 用途 |
|---------|-----------|------|
| `audit_validation.budget_schema` | `usage_account_workspace_name` | ワークスペースID・名前のマスタ |
| `audit_validation.budget_schema` | `account_budget_contracts` | 契約・予算設定情報 |
| `audit_validation.usage_schema` | `ws_usage_filtered` | 利用料データ（フィルタ済み） |

### 2.2 主要テーブル詳細

#### 2.2.1 `usage_account_workspace_name`
ワークスペースのマスタテーブル。同名のワークスペースが複数存在する場合、`create_time`が最新のものを使用。

| カラム名 | 説明 |
|---------|------|
| `workspace_id` | ワークスペースID |
| `workspace_name` | ワークスペース名 |
| `create_time` | 作成日時 |

#### 2.2.2 `account_budget_contracts`
予算・契約情報を管理するテーブル。

| カラム名 | 説明 |
|---------|------|
| `contract_id` | 契約ID |
| `workspace_id` | ワークスペースID（個別指定時） |
| `scope_type` | スコープタイプ（`ACCOUNT`/`WORKSPACE`） |
| `budget_usd` | 予算金額（USD） |
| `threshold_usd` | 閾値金額（USD） |
| `contract_start_date` | 契約開始日 |
| `contract_end_date` | 契約終了日 |
| `active` | 有効フラグ |
| `updated_at` | 更新日時 |

#### 2.2.3 `ws_usage_filtered`
フィルタリング済みの利用料データ。

| カラム名 | 説明 |
|---------|------|
| `workspace_id` | ワークスペースID |
| `usage_date` | 利用日 |
| `usage_usd` | 利用金額（USD） |
| `sku_name` | SKU名 |
| `billing_origin_product` | 請求元製品 |

---

## 3. パラメータ定義

### 3.1 グローバルパラメータ

| パラメータ名 | 表示名 | データ型 | デフォルト値 | 説明 |
|-------------|-------|---------|-------------|------|
| `param_start_date` | 開始日 | DATE | `now-12M/M`〜`now-6M/M` | 集計開始日 |
| `param_end_date` | 終了日 | DATE | `now/d`〜`now-1d/d` | 集計終了日 |
| `param_workspace` | 対象ワークスペース | STRING | `<ALL WORKSPACES>` | ワークスペースの絞り込み |
| `param_time_key` | 期間粒度 | STRING | `日`/`週`/`月` | 集計粒度の選択 |
| `param_group_key` | 絞り込み対象 | STRING | `ワークスペース別` | グルーピングキーの選択 |
| `param_contract_start_date` | 契約開始日 | DATE | `now-12M/M` | 契約期間の開始日（オプション） |

### 3.2 期間粒度の選択肢
- **日**: 日単位での集計
- **週**: 週単位での集計（週の開始日でグルーピング）
- **月**: 月単位での集計

### 3.3 絞り込み対象の選択肢
- **ワークスペース別**: ワークスペース単位での集計
- **請求元製品別**: billing_origin_product単位での集計
- **SKU別**: sku_name単位での集計

### 3.4 ワークスペースパラメータの特殊値
以下の値は「全ワークスペース」として扱われる：
- `<ALL WORKSPACES>`
- `ALL WORKSPACES`
- `ALL WORKSPACE`
- `<ALL WORKSPACE>`
- `全ワークスペース`
- `すべて`
- `ALL-WORKSPACE`

---

## 4. データセット定義

### 4.1 `select_time_key_overview`
期間粒度選択用のデータセット。

**出力カラム**: `time_key`（日/週/月）

### 4.2 `select_group_key`
グルーピングキー選択用のデータセット。

**出力カラム**: `group_key`（ワークスペース別/請求元製品別/SKU別）

### 4.3 `select_workspace`
ワークスペース選択用のデータセット。

**出力カラム**: `workspace_name`

### 4.4 `budget_vs_actual` / `budget_vs_actual_260115`
予算対実績データを取得するデータセット。

**主要な出力カラム**:
| カラム名 | 説明 |
|---------|------|
| `期間` | 表示用の期間文字列 |
| `当期間利用料_USD` | 当該期間の利用料 |
| `契約開始からの累積利用料_USD` | 累積利用料 |
| `予算閾値_USD` | 閾値金額 |
| `予算値_USD` | 予算金額 |
| `閾値超過有無` | 閾値超過フラグ（BOOLEAN） |
| `予算超過有無` | 予算超過フラグ（BOOLEAN） |
| `閾値超過額_USD` | 閾値超過金額 |
| `予算超過額_USD` | 予算超過金額 |

### 4.5 `billing_originproduct_and_sku_graph`
請求元製品・SKU別のグラフ用データセット。

**主要な出力カラム**:
| カラム名 | 説明 |
|---------|------|
| `display_date` | 表示用日付 |
| `group_key` | グルーピングキー |
| `usage_usd` | 利用金額（USD） |

**特記事項**:
- ワークスペース数が50を超える場合、Top 10以外は`<OTHERS>`にまとめられる

### 4.6 `billing_originproduct_and_sku_table`
請求元製品・SKU別のテーブル用データセット。

**主要な出力カラム**:
| カラム名 | 説明 |
|---------|------|
| `group_key` | 種別 |
| `Start to End date` | 集計期間合計 |
| `Current period` | 当期間 |
| `Last period` | 1期間前 |
| `2 periods ago`〜`5 periods ago` | 2〜5期間前 |

**特記事項**:
- 各期間の変化率（%）も表示
- TOTAL行が自動で追加される

### 4.7 `cumulative_usage` / `cumulative_usage_table`
累積利用料データセット。

**主要な出力カラム**:
| カラム名 | 説明 |
|---------|------|
| `display_date` | 表示用日付 |
| `cumulative_usage_usd` | 累積利用料（USD） |
| `total_budget_usd` | 予算金額 |
| `total_threshold_usd` | 閾値金額 |

### 4.8 `total_usage`
期間ごとの利用料データセット。

**主要な出力カラム**:
| カラム名 | 説明 |
|---------|------|
| `display_date` | 表示用日付 |
| `period_usage_usd` | 期間利用料（USD） |

---

## 5. ウィジェット構成

### 5.1 フィルターウィジェット

| ウィジェット名 | タイプ | 表示名 | 位置（x,y） | サイズ |
|--------------|--------|-------|------------|--------|
| `globalStartDate` | 日付ピッカー | 開始日 | (0, 0) | 2×1 |
| `globalEndDate` | 日付ピッカー | 終了日 | (0, 1) | 2×1 |
| `s1ParamTimeKey` | 単一選択 | 期間粒度 | (2, 1) | 2×1 |
| `8a44df86` | 単一選択 | 絞り込み対象 | (4, 1) | 2×1 |
| `workspace` | 単一選択 | 対象ワークスペース | (2, 0) | 3×1 |

### 5.2 グラフウィジェット

| ウィジェット名 | タイプ | タイトル | 位置（x,y） | サイズ |
|--------------|--------|---------|------------|--------|
| `jissekichi-gurafu` | 棒グラフ | 実績値グラフ(USD) | (0, 2) | 6×6 |
| `ruiseki-riyouryou-gurafu` | コンボグラフ | 累積利用料グラフ(USD) | (0, 8) | 6×6 |
| `seikyumoto-seihinbetsu-sku-betsu-gurafu-shuuseiban-2` | 積み上げ棒グラフ | ワークスペース別・請求元製品別・SKU別グラフ(USD) | (0, 20) | 6×6 |

### 5.3 テーブルウィジェット

| ウィジェット名 | タイトル | 位置（x,y） | サイズ |
|--------------|---------|------------|--------|
| `yojitsu-kanri-teburu-shuuseiban` | 予実管理テーブル(USD) | (0, 14) | 6×6 |
| `seikyumoto-seihinbetsu-sku-betsu-teburu-1` | ワークスペース別・請求元製品別・SKU別テーブル(USD) | (0, 26) | 6×5 |

---

## 6. ウィジェット詳細

### 6.1 実績値グラフ(USD)
- **データセット**: `total_usage`
- **タイプ**: 棒グラフ
- **X軸**: 期間（`display_date`）
- **Y軸**: 期間利用料（`period_usage_usd`）、通貨形式（USD）
- **特徴**: 選択した期間粒度に応じた実績値を表示

### 6.2 累積利用料グラフ(USD)
- **データセット**: `cumulative_usage`
- **タイプ**: コンボグラフ
- **X軸**: 期間（`display_date`）
- **主Y軸（棒）**: 累積利用料（`cumulative_usage_usd`）
- **副Y軸（線）**: 予算（`total_budget_usd`）、閾値（`total_threshold_usd`）
- **特徴**: 累積利用料と予算/閾値のラインを同時表示

### 6.3 予実管理テーブル(USD)
- **データセット**: `budget_vs_actual_260115`
- **表示カラム**:
  1. 期間
  2. 期間合計利用料
  3. 期間累積利用料
  4. 予算
  5. 予算閾値
  6. 予算超過フラグ
  7. 閾値超過フラグ
  8. 予算超過料
  9. 閾値超過料
- **ページネーション**: 25件/ページ
- **コンデンス表示**: 有効

### 6.4 ワークスペース別・請求元製品別・SKU別グラフ(USD)
- **データセット**: `billing_originproduct_and_sku_graph`
- **タイプ**: 積み上げ棒グラフ
- **X軸**: 期間（`display_date`）
- **Y軸**: 利用料（`usage_usd`）
- **色分け**: グルーピングキー（`group_key`）
- **特徴**: SKU別にカラー分けされた積み上げ棒グラフ

### 6.5 ワークスペース別・請求元製品別・SKU別テーブル(USD)
- **データセット**: `billing_originproduct_and_sku_table`
- **表示カラム**:
  1. 種別（グルーピングキー）
  2. 集計期間（Start to End date）
  3. 当期間〜5期間前の利用料（変化率付き）
- **特徴**: 
  - HTML形式で変化率を色分け表示（+10%超: 緑、-10%超: 赤）
  - TOTAL行が自動追加
- **ページネーション**: 25件/ページ

---

## 7. ビジネスロジック

### 7.1 ワークスペース一意化ロジック
同名のワークスペースが複数存在する場合の対応：
```sql
ROW_NUMBER() OVER (
  PARTITION BY workspace_name
  ORDER BY create_time DESC
) AS rn
-- rn = 1 のレコードのみ使用
```

### 7.2 予算設定選択ロジック
複数の予算設定が存在する場合の優先順位：
1. `param_workspace`が個別指定の場合、該当ワークスペースの予算を優先
2. 同一条件内では、`updated_at`が最新のものを使用

### 7.3 利用料集計ロジック
- **日単位**: `usage_date`そのまま
- **週単位**: `date_trunc('WEEK', usage_date)`
- **月単位**: `date_trunc('MONTH', usage_date)`

### 7.4 累積計算ロジック
```sql
SUM(usage_usd) OVER (
  PARTITION BY workspace_id
  ORDER BY period
  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
) AS cumulative_usd
```

### 7.5 変化率計算ロジック
```sql
round(
  try_divide(
    usage_usd - prev_usage_usd,
    prev_usage_usd
  ) * 100,
  2
) AS usage_change_percentage
```

---

## 8. 表示形式

### 8.1 日付表示形式
| 粒度 | 形式 | 例 |
|-----|------|-----|
| 日 | `yyyy年MM月dd日` | 2026年01月28日 |
| 週 | `yyyy年MM月dd日〜MM月dd日` | 2026年01月20日〜01月26日 |
| 月 | `yyyy年MM月` | 2026年01月 |

### 8.2 金額表示形式
- **グラフ**: コンパクト形式（K/M/B）、最大小数点2桁
- **テーブル**: 数値形式、適宜フォーマット

### 8.3 変化率の色分け
| 条件 | 色コード |
|-----|---------|
| +10%超 | `#00A972`（緑） |
| -10%超 | `#FF3621`（赤） |
| その他 | `#919191`（グレー） |

---

## 9. UI設定

### 9.1 レイアウト
- **ページタイプ**: キャンバス（`PAGE_TYPE_CANVAS`）
- **グリッドサイズ**: 6列構成
- **テーマ**: デフォルト

### 9.2 適用モード
- `applyModeEnabled`: false（即時反映）

---

## 10. SKU一覧（参考）

ダッシュボードで表示されるSKUの例：
- ENTERPRISE_ALL_PURPOSE_COMPUTE
- ENTERPRISE_ALL_PURPOSE_COMPUTE_(PHOTON)
- ENTERPRISE_ALL_PURPOSE_SERVERLESS_COMPUTE_AP_TOKYO
- ENTERPRISE_ANTHROPIC_MODEL_SERVING
- ENTERPRISE_DLT_ADVANCED_COMPUTE
- ENTERPRISE_GEMINI_MODEL_SERVING
- ENTERPRISE_JOBS_COMPUTE
- ENTERPRISE_JOBS_COMPUTE_(PHOTON)
- ENTERPRISE_JOBS_SERVERLESS_COMPUTE_AP_TOKYO
- ENTERPRISE_SERVERLESS_REAL_TIME_INFERENCE_AP_TOKYO
- ENTERPRISE_SERVERLESS_SQL_COMPUTE_AP_TOKYO
- ENTERPRISE_SQL_COMPUTE
- ENTERPRISE_SQL_PRO_COMPUTE_AP_TOKYO
- INTER_AVAILABILITY_ZONE_EGRESS
- INTER_REGION_EGRESS_FROM_AP_TOKYO
- INTERNET_EGRESS_FROM_AP_TOKYO
- PUBLIC_CONNECTIVITY_DATA_PROCESSED_AP_TOKYO

---

## 11. 変更履歴

| 日付 | バージョン | 変更内容 |
|-----|-----------|---------|
| 2026-01-28 | 1.0 | 初版作成 |
