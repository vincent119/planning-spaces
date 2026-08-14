# Report BI Templates Migration 與回滾說明

## 結論

本次 BI 範本改造不需要新增 database migration。

原因如下：

- `report_templates.config` 已是 `JSONB NOT NULL`，可同時保存 V1 與 V2 config。
- V2 config 以 `version: 2` 作為版本辨識；缺少 `version` 或 `version = 1` 的資料由 server 視為 V1 legacy。
- API response 的 `config_version` 與 `config_legacy` 由 repository 讀取 `config` 後動態解析，不需要實體欄位。
- 範本列表所需 `dataset`、X 軸、series、metric、chart type 可由 `config` 直接取得，目前沒有跨範本搜尋或排序這些欄位的需求。
- 既有 active index 與 unique index 已由 `0030_align_report_templates.sql` 建立，V2 不改變查詢條件。

## 欄位評估

| 項目 | 是否新增 | 理由 |
| ---- | ---- | ---- |
| `config_version` | 否 | 可從 `config->>'version'` 或 server parser 推導；V1 缺少 version 需保留 legacy 語意 |
| `config_legacy` | 否 | legacy 是相容策略結果，不是使用者輸入資料；由 server parser 判斷較不易失真 |
| `dataset` | 否 | MVP 只有 `ticket_events`，列表顯示可從 V2 config 取得 |
| `x_axis` / `series` / `metric` / `chart_type` | 否 | 目前僅用於列表顯示，不作 DB filter / sort |
| metadata 加速欄位 | 否 | 尚未有大量範本查詢與後端排序需求，新增欄位會增加 backfill 與同步成本 |

## 未來何時需要 migration

若後續出現以下需求，再新增 migration：

- 後端需要依 `dataset`、`config_version`、`chart_type` 篩選或排序範本。
- 範本數量成長到前端解析 config 造成可感知延遲。
- 需要在 SQL 層建立報表範本管理頁的搜尋索引。
- 需要審計或治理報表直接統計 V1 / V2 範本數量。

建議未來 migration 可採用 nullable 或 generated 欄位：

```sql
ALTER TABLE report_templates
  ADD COLUMN config_version SMALLINT,
  ADD COLUMN dataset VARCHAR(64);

UPDATE report_templates
SET
  config_version = COALESCE((config->>'version')::SMALLINT, 1),
  dataset = NULLIF(config->>'dataset', '');
```

若要避免資料同步問題，也可使用 expression index：

```sql
CREATE INDEX idx_report_templates_config_version_active
  ON report_templates ((COALESCE((config->>'version')::SMALLINT, 1)))
  WHERE deleted_at IS NULL;
```

## 回滾說明

### 回滾原則

- 不刪除既有 V1 範本。
- 不刪除已建立的 V2 範本資料。
- V2 config 可保留在 `report_templates.config`，但舊版程式若不認得 `version = 2`，可能無法編輯或執行。
- 回滾應優先回滾應用程式，資料保持原狀。

### 回滾到不支援 V2 的版本

若系統回滾到尚未支援 `TemplateConfigV2` 的版本：

- V1 範本可繼續使用。
- V2 範本仍保存在 DB，但舊版程式可能在 unmarshal 或執行時失敗。
- 管理員可暫時隱藏或避免操作 V2 範本，直到程式升回支援版本。

### 若必須暫停 V2 範本

不建議刪除資料。若營運上必須暫停 V2 範本，可使用軟刪除保留資料：

```sql
UPDATE report_templates
SET deleted_at = NOW(), updated_at = NOW()
WHERE deleted_at IS NULL
  AND config->>'version' = '2';
```

復原時：

```sql
UPDATE report_templates
SET deleted_at = NULL, updated_at = NOW()
WHERE config->>'version' = '2';
```

執行前需先確認這些範本名稱不會與 active 範本的 `(project_id, name)` unique index 衝突。

## 驗證方式

檢查 V1 / V2 範本分布：

```sql
SELECT
  COALESCE(config->>'version', '1') AS config_version,
  COUNT(*) AS count
FROM report_templates
WHERE deleted_at IS NULL
GROUP BY COALESCE(config->>'version', '1')
ORDER BY config_version;
```

檢查 V2 範本是否含必要欄位：

```sql
SELECT id, project_id, name
FROM report_templates
WHERE deleted_at IS NULL
  AND config->>'version' = '2'
  AND (
    config->>'dataset' IS NULL
    OR config->'query'->>'x_axis' IS NULL
    OR config->'query'->>'metric' IS NULL
    OR config->'visualization'->>'chart_type' IS NULL
  );
```
