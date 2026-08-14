# 三班固定制排班與假勤管理後端設計

## 設計原則

- 排班 / 假勤為獨立 bounded context，建議 package 命名為 `internal/schedule`。
- 使用者帳號不保存班別欄位。
- 第一階段只支援三班固定制，不支援三班輪班。
- 人員歸屬需綁定到具體 `duty_shifts`，不能只綁 `shift_groups`。
- 工程師轉班只能建立新的生效區間，不得覆蓋歷史班別歸屬。
- 歷史報表查班別時必須依事件日期回查當時有效歸屬，不得使用使用者目前班別。
- 排班週期建立後自動展開初稿，初稿所有日期皆為 `workday`，後續主要編輯 `day_type`。
- 排班週期與確認檢查需依 `4week` / `8week` 變形工時模式執行。
- 所有日期以 Asia/Taipei 業務日期處理。
- Ticket 不依賴本模組建立或更新；未來僅透過查詢整合。

## 系統設定

新增或沿用 Global Setting：

```text
schedule.labor_law_mode = 4week | 8week
schedule.min_staff_per_core_shift = integer
schedule.current_shift_group_code = string
schedule.public_holiday_source = manual | api
schedule.public_holiday_provider = data_gov_tw_123662 | taiwan_calendar_json
```

語意：

- `4week`：預設週期 28 天。
- `8week`：預設週期 56 天。
- 建立週期時需將當下模式快照寫入 `shift_periods.labor_law_mode`，避免日後全域設定變更影響既有週期與歷史報表。
- `schedule.min_staff_per_core_shift`：早班 / 中班 / 晚班共用的最小值班人數，預設值為 `1`，最小值為 `1`。
- 最小值班人數只套用 `3shift_fixed` 內的核心班別，第一階段以 `duty_shifts.code IN ('morning', 'afternoon', 'night')` 判斷；其他班別不套用此限制。
- `schedule.current_shift_group_code`：現行值班班制代碼，預設值為 `3shift_fixed`。Ticket 列表、Report 與排班相關班別選項需透過此設定解析目前有效班制，不得在前端硬編碼 `3shift_fixed`。
- 讀取 `schedule.current_shift_group_code` 時需驗證對應 `shift_groups.code` 存在且 `is_active = TRUE`；若設定缺失可由後端以 `3shift_fixed` 作為相容 fallback，但需以 seed 補齊設定，前端不得自行 fallback 為全部班別。
- 更新 `schedule.current_shift_group_code` 時需驗證新值對應啟用中的 `shift_groups.code`；驗證失敗需拒絕儲存並回傳可診斷錯誤。
- `schedule.public_holiday_source`：國定假日資料來源，`manual` 表示只使用系統內手動維護資料，`api` 表示允許管理者或排程呼叫官方資料來源同步。預設 `manual`。
- `schedule.public_holiday_provider`：官方資料 adapter 代碼，預設 `data_gov_tw_123662`。可選 `taiwan_calendar_json` 作為 JSON/CDN 備援來源。後端需以 adapter 隔離外部格式，不得讓外部欄位直接散落在排班服務；adapter 輸出給排班使用的資料只保留特殊國定假日，普通週六、週日不得輸出成排班用國假。
- Global Setting 修改權限沿用系統設定管理權限，僅 `admin` / `op_admin` 可調整。

建議 seed：

```sql
INSERT INTO system_settings (key, value, category, value_type, description, sort_order, is_secret)
VALUES (
  'schedule.current_shift_group_code',
  '3shift_fixed',
  'system',
  'string',
  '現行值班班制代碼，Ticket 篩選、Report 班別選項與排班歸屬預設依此班制解析',
  15,
  FALSE
)
ON CONFLICT (key) DO NOTHING;

INSERT INTO system_settings (key, value, category, value_type, description, sort_order, is_secret)
VALUES
  (
    'schedule.public_holiday_source',
    'manual',
    'system',
    'string',
    '國定假日來源：manual（手動維護）、api（允許同步官方資料）',
    16,
    FALSE
  ),
  (
    'schedule.public_holiday_provider',
    'data_gov_tw_123662',
    'system',
    'string',
    '國定假日官方資料 provider：data_gov_tw_123662 或 taiwan_calendar_json',
    17,
    FALSE
  )
ON CONFLICT (key) DO NOTHING;
```

## 資料模型

### shift_groups

班制定義。本階段 seed 一筆 `3shift_fixed`。

```sql
CREATE TABLE shift_groups (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  code VARCHAR(32) NOT NULL UNIQUE,
  name VARCHAR(64) NOT NULL,
  type VARCHAR(32) NOT NULL CHECK (type IN ('fixed')),
  description TEXT,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

INSERT INTO shift_groups (code, name, type, description)
VALUES ('3shift_fixed', '三班固定制', 'fixed', '早 / 中 / 晚固定班，不輪班');
```

### duty_shifts

具體班別。第一階段 seed 早班、中班、晚班。

```sql
CREATE TABLE duty_shifts (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  group_id CHAR(26) NOT NULL REFERENCES shift_groups(id),
  code VARCHAR(32) NOT NULL UNIQUE,
  name VARCHAR(64) NOT NULL,
  start_time TIME NOT NULL,
  end_time TIME NOT NULL,
  crosses_midnight BOOLEAN NOT NULL DEFAULT FALSE,
  sort_order SMALLINT NOT NULL DEFAULT 0,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_duty_shifts_group_sort ON duty_shifts (group_id, sort_order);
```

建議 seed：

```text
morning    早班 07:00-15:00 crosses_midnight=false
afternoon  中班 15:00-23:00 crosses_midnight=false
night      晚班 23:00-07:00 crosses_midnight=true
```

### user_shift_assignments

人員固定班別歸屬。此表必須綁定 `duty_shift_id`。

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE user_shift_assignments (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  user_id CHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  shift_group_id CHAR(26) NOT NULL REFERENCES shift_groups(id),
  duty_shift_id CHAR(26) NOT NULL REFERENCES duty_shifts(id),
  effective_from DATE NOT NULL,
  effective_to DATE,
  remark TEXT,
  assigned_by CHAR(26) NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT chk_assignment_range CHECK (effective_to IS NULL OR effective_to > effective_from),
  CONSTRAINT uq_user_shift_assignment_no_overlap EXCLUDE USING gist (
    user_id WITH =,
    daterange(effective_from, COALESCE(effective_to, '9999-12-31'::DATE), '[)') WITH &&
  )
);

CREATE INDEX idx_user_shift_assignments_user_range
  ON user_shift_assignments (user_id, effective_from, effective_to);
CREATE INDEX idx_user_shift_assignments_shift_date
  ON user_shift_assignments (duty_shift_id, effective_from, effective_to);
```

App layer 需驗證 `duty_shift_id` 屬於 `shift_group_id`。

#### 轉班規則

轉班不得直接更新既有歸屬的 `duty_shift_id`。例如工程師從早班轉晚班，應執行：

```text
原歸屬：早班 effective_from=2026-05-01 effective_to=NULL
轉班日：2026-07-01

更新原歸屬 effective_to=2026-07-01
新增新歸屬 晚班 effective_from=2026-07-01 effective_to=NULL
```

此模型確保：

- 查詢 2026-06 的報表仍得到早班。
- 查詢 2026-07 之後的報表才得到晚班。
- confirmed / locked 週期不會因人員目前班別異動而被重算。

歷史修正若涉及已確認週期或已產生報表期間，需使用具審計紀錄的校正流程，不允許靜默覆蓋。

### shift_periods

排班週期。一個週期包含早 / 中 / 晚三個分組。

```sql
CREATE TABLE shift_periods (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  name VARCHAR(64) NOT NULL,
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  shift_group_id CHAR(26) NOT NULL REFERENCES shift_groups(id),
  labor_law_mode VARCHAR(16) NOT NULL CHECK (labor_law_mode IN ('4week','8week')),
  status VARCHAR(16) NOT NULL DEFAULT 'draft'
    CHECK (status IN ('draft','confirmed','locked')),
  skipped_users JSONB NOT NULL DEFAULT '[]'::JSONB,
  remark TEXT,
  created_by CHAR(26) NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT chk_shift_period_range CHECK (end_date >= start_date)
);

CREATE INDEX idx_shift_periods_group_start ON shift_periods (shift_group_id, start_date);
CREATE INDEX idx_shift_periods_status ON shift_periods (status);
```

`skipped_users` 用來保存自動展開時被略過的人員與原因，例如停用帳號、沒有有效班別歸屬。

### user_schedules

每日排班主表。初稿展開後主要修改 `day_type`。

```sql
CREATE TABLE user_schedules (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  period_id CHAR(26) NOT NULL REFERENCES shift_periods(id) ON DELETE CASCADE,
  user_id CHAR(26) NOT NULL REFERENCES users(id),
  duty_shift_id CHAR(26) NOT NULL REFERENCES duty_shifts(id),
  schedule_date DATE NOT NULL,
  day_type VARCHAR(20) NOT NULL DEFAULT 'workday'
    CHECK (day_type IN (
      'workday',
      'statutory_holiday',
      'day_off',
      'public_holiday',
      'annual',
      'official',
      'sick',
      'personal',
      'other'
    )),
  source VARCHAR(16) NOT NULL DEFAULT 'auto'
    CHECK (source IN ('auto','manual')),
  leave_source VARCHAR(16) NOT NULL DEFAULT 'manual'
  CHECK (leave_source IN ('manual','hr_system')),
  remark TEXT,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_by CHAR(26) NOT NULL REFERENCES users(id),
  updated_by CHAR(26) REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX ux_user_schedules_period_user_date
  ON user_schedules (period_id, user_id, schedule_date)
  WHERE is_active = TRUE;
CREATE INDEX idx_user_schedules_period_shift_date
  ON user_schedules (period_id, duty_shift_id, schedule_date)
  WHERE is_active = TRUE;
CREATE INDEX idx_user_schedules_user_date
  ON user_schedules (user_id, schedule_date)
  WHERE is_active = TRUE;
```

### public_holidays

國定假日與政府辦公日曆資料表。此表保存同步後的內部模型，不直接等同於某個週期已排定的 `public_holiday` day type。

```sql
CREATE TABLE public_holidays (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  holiday_date DATE NOT NULL,
  year SMALLINT NOT NULL,
  name VARCHAR(64) NOT NULL,
  is_holiday BOOLEAN NOT NULL DEFAULT TRUE,
  source VARCHAR(16) NOT NULL CHECK (source IN ('manual','api')),
  provider VARCHAR(32) NOT NULL DEFAULT 'manual',
  source_key VARCHAR(128),
  raw_payload JSONB,
  synced_at TIMESTAMPTZ,
  created_by CHAR(26) REFERENCES users(id),
  updated_by CHAR(26) REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

CREATE UNIQUE INDEX ux_public_holidays_active_date
  ON public_holidays (holiday_date)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_public_holidays_year
  ON public_holidays (year)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_public_holidays_source
  ON public_holidays (source, provider)
  WHERE deleted_at IS NULL;
```

同步規則：

- `holiday_date` 為業務唯一鍵；同一日期重複同步只能 update，不可新增重複資料。
- `source = manual` 的資料預設優先，API 同步遇到同日期 manual 資料時保留 manual 值，並在 sync result 標示 `skipped_manual_override`。
- `is_holiday = TRUE` 代表排班用特殊國定假日；官方資料若只是普通週六 / 週日、補班或上班日，需保存為 `is_holiday = FALSE` 或略過，不得錯當國定假。
- `raw_payload` 只供診斷與審計，不作為業務查詢來源。
- API 同步失敗不得清空既有年度資料；回傳錯誤並保留原資料。

### public_holiday_sync_runs

國定假日同步執行紀錄。

```sql
CREATE TABLE public_holiday_sync_runs (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  year SMALLINT NOT NULL,
  provider VARCHAR(32) NOT NULL,
  mode VARCHAR(16) NOT NULL CHECK (mode IN ('preview','apply')),
  status VARCHAR(16) NOT NULL CHECK (status IN ('success','failed','partial')),
  fetched_count INTEGER NOT NULL DEFAULT 0,
  inserted_count INTEGER NOT NULL DEFAULT 0,
  updated_count INTEGER NOT NULL DEFAULT 0,
  skipped_count INTEGER NOT NULL DEFAULT 0,
  error_message TEXT,
  diff_summary JSONB,
  triggered_by CHAR(26) REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

`preview` 模式只寫入 sync run，不更新 `public_holidays`。`apply` 模式需在同一交易中寫入差異與資料更新，並記錄審計。

### user_schedule_history

排班異動審計。

```sql
CREATE TABLE user_schedule_history (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  schedule_id CHAR(26) NOT NULL,
  period_id CHAR(26) NOT NULL,
  user_id CHAR(26) NOT NULL,
  duty_shift_id CHAR(26) NOT NULL,
  schedule_date DATE NOT NULL,
  action VARCHAR(16) NOT NULL CHECK (action IN ('INSERT','UPDATE','DELETE')),
  changed_by CHAR(26) REFERENCES users(id),
  changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  before_snapshot JSONB,
  after_snapshot JSONB
);

CREATE INDEX idx_user_schedule_history_schedule
  ON user_schedule_history (schedule_id, changed_at DESC);
CREATE INDEX idx_user_schedule_history_user_date
  ON user_schedule_history (user_id, schedule_date, changed_at DESC);
```

### shift_period_confirmations

週期確認。

```sql
CREATE TABLE shift_period_confirmations (
  id CHAR(26) PRIMARY KEY DEFAULT generate_ulid(),
  period_id CHAR(26) NOT NULL REFERENCES shift_periods(id) ON DELETE CASCADE,
  confirmed_by CHAR(26) NOT NULL REFERENCES users(id),
  confirmed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  remark TEXT,
  UNIQUE (period_id, confirmed_by)
);

CREATE INDEX idx_shift_period_confirmations_period
  ON shift_period_confirmations (period_id);
```

## 週期初稿展開

建立週期時同一 transaction 內執行：

1. 建立 `shift_periods`。
2. 查詢週期內有效且啟用的人員班別歸屬。
3. 對每位有效人員與每一天建立 `user_schedules`。
4. `duty_shift_id` 使用該日有效歸屬。
5. `day_type = workday`。
6. `source = auto`。
7. 寫入 `user_schedule_history`，action 為 `INSERT`。
8. 回傳展開摘要與 skipped users。

初稿展開不得依星期或行事曆標記自動改格：

- 週六、週日仍建立為 `workday`。
- `calendar_marks` 內的特殊國定假日仍建立為 `workday`。
- 工程師或值班組長後續可手動將格子改為 `statutory_holiday`、`day_off` 或 `public_holiday`。
- 重新整理 draft 週期採補缺策略時也遵守相同規則，不覆蓋既有手動編輯。

若人員在週期中途換班，展開需依日期使用當日有效班別。

若轉班發生於週期建立後：

- `confirmed` / `locked` 週期：不自動更新既有 `user_schedules`。
- `draft` 週期：可由值班組長選擇是否重算該人轉班日起的未來日期；重算需寫入 `user_schedule_history`。
- 新週期：依最新有效歸屬自動展開。

本階段重建 draft 採「補缺」策略：只新增週期內目前缺少、且當日已有有效人員班別歸屬的 `user_schedules`，不覆蓋既有手動編輯格。重建後需更新 `skipped_users`，並寫入週期操作審計與新增排班歷史。

## Report 查詢契約

本 BC 提供依人員與日期查詢當時有效班別的查詢能力，供未來 Ticket / Report 使用：

```text
ResolveUserDutyShift(user_id, event_date) -> duty_shift
ResolveUsersDutyShift(user_ids, event_date) -> map[user_id]duty_shift
```

查詢條件必須使用事件日期：

```sql
SELECT usa.user_id, ds.id, ds.code, ds.name
FROM user_shift_assignments usa
JOIN duty_shifts ds ON ds.id = usa.duty_shift_id
WHERE usa.user_id = $1
  AND usa.effective_from <= $2
  AND (usa.effective_to IS NULL OR usa.effective_to > $2);
```

### Ticket / Report 班別查詢整合

Ticket 列表不新增 Ticket schema 欄位。列表 API 追加 `duty_shift_id` 查詢參數，語意為「Ticket 建立者在 Ticket 建立日期的有效班別」。列表 response 可回傳 `creator_duty_shift_id`、`creator_duty_shift_code`、`creator_duty_shift_name` 作為查詢投影。

Ticket / Report 班別選項需先讀取 `schedule.current_shift_group_code`，解析到現行 `shift_groups.id` 後，只回傳該班制底下啟用的 `duty_shifts`。不得回傳所有班制的啟用班別，避免不同班制中同名的早班 / 中班 / 晚班在 UI 重覆出現。收到 `duty_shift_id` 篩選時，後端需驗證該班別屬於現行值班班制；不符合時回傳 400。

Report `ticket_events` 資料集追加 `shift` 維度與篩選：

- `person_basis = created_by`：以 `tickets.created_by` 與 `tickets.created_at` 的 Asia/Taipei 日期解析班別。
- `person_basis = actor`：以 `ticket_activities.actor_id` 與 `ticket_activities.created_at` 的 Asia/Taipei 日期解析班別。
- 查無有效班別時顯示「未歸屬班別」。

每日值班執行統計的班別列也使用上述解析結果，不得以事件發生時段推估早 / 中 / 晚。

Report 禁止使用「目前有效班別」回填歷史事件，避免轉班後改變既有統計結果。

## 驗證規則

### 規則引擎分層

第一階段不實作完整勞基法規則引擎，但需將排班驗證切分成可擴充的規則：

```text
staffing_rule    營運人力規則，例如早中晚班最小值班人數
labor_rule       勞基法規則，例如四週 / 八週例假與休息日檢查
validation_phase single_cell_update | refresh_draft | confirm_period
severity         block | warning | info
```

目前實作範圍：

- `staffing_rule` 在單格儲存與確認週期時都必須以 `block` 執行。
- `labor_rule` 第一階段保留既有四週 / 八週例假缺漏檢查，單格儲存可回傳 warning，確認週期時以 `block` 執行。
- 完整勞基法規則引擎仍列為 deferred task，不在本階段一次補完。

### 週期狀態

- `draft` 可修改。
- `confirmed` 不可修改。
- `locked` 不可修改。

### 早中晚班最小值班人數

舊規則「同班同日只能一人排例假 / 休假」需被最小值班人數規則取代。儲存單格時以更新後狀態檢查：

```sql
SELECT COUNT(*)
FROM user_schedules
WHERE period_id = $1
  AND duty_shift_id = $2
  AND schedule_date = $3
  AND day_type = 'workday'
  AND is_active = TRUE
  AND id <> $4;
```

檢查規則：

- 僅套用 `duty_shifts.code IN ('morning', 'afternoon', 'night')`。
- `day_type = 'workday'` 視為值班。
- 其他 `day_type` 均視為非值班，包含 `statutory_holiday`、`day_off`、`public_holiday`、`annual`、`official`、`sick`、`personal`、`other`。
- 若新值為非 `workday`，需確認更新後同一 `period_id + duty_shift_id + schedule_date` 的 `workday` 人數仍大於等於 `schedule.min_staff_per_core_shift`。
- 若新值為 `workday`，必須允許，因為會增加值班人數。
- 非核心班別不做最小值班人數阻擋。
- 違反時回傳 409 與可診斷訊息，例如「無法排休：此班別當日需至少保留 1 位值班人員」。

確認週期時需重新掃描早班 / 中班 / 晚班所有日期，若任一天低於最小值班人數，回傳 409 並列出日期、班別、目前值班人數與需求人數。

### 四週 / 八週變形工時例假與休假檢查

每人依週期 `labor_law_mode` 執行例假與休假檢查：

- `4week`：以 28 天為主要檢查週期。
- `8week`：以 56 天為主要檢查週期。

週檢查區間以 `shift_periods.start_date` 為起點，每 7 天切分一次；4 週週期應有 4 個週檢查區間，8 週週期應有 8 個週檢查區間。後續若要擴充完整勞基法規則，引擎需以 `labor_law_mode` 決定檢查規則集。

例假規則：

- 每位工程師每個週檢查區間必須剛好 1 天 `statutory_holiday`。
- 若同一週檢查區間為 0 天，視為缺例假。
- 若同一週檢查區間超過 1 天，視為例假超排。
- 驗收口徑需對齊既有 Excel 班表公式：週期起日後每 7 天為一段，逐段檢查 `COUNTIF(區間, "*例*") = 1`。以 4 週週期為例，若畫面欄位對應 Excel `B:AC`，則檢查區間為 `B:H`、`I:O`、`P:V`、`W:AC`。
- 單格更新若會讓同一週檢查區間超過 1 天 `statutory_holiday`，需以 409 阻擋。
- 單格更新若只是移除既有例假造成缺例假，可回傳 warning，不阻擋；確認週期時必須阻擋。

休假規則：

- `day_off` 不預設綁定週六或週日，初稿所有週末仍為 `workday`。
- 應排休假數以週檢查區間數計算；4 週為 4 天，8 週為 8 天。
- 國定假日、補假日或已有補假日的原始國定假日不得改變 `day_off.target_count`；若原始國定假日被使用者排為 `day_off`，只計入 `day_off.used_count`。
- 休假可在週期內移動日期，不要求每個週檢查區間剛好 1 天。
- 確認週期時需檢查每人 `day_off` 已排數量與應排數量一致。

國定假規則：

- 應排國定假數以週期範圍內 `public_holidays.is_holiday = TRUE` 且不是週日、不是已有補假日之原週六假日的特殊國定假日數量計算。
- 普通週六、週日不得列入 `public_holiday` 應排數。
- `public_holiday` 格子由使用者排定，系統不因日期標記自動改格。
- `public_holiday` 可在週期內任意日期排定，不綁定 `calendar_marks.is_balance_date`；行事曆標記只決定 `public_holiday.target_count` 與日期欄顯示。
- `public_holiday.used_count` 只精準計算 `day_type = public_holiday` 的格子，不因餘額日同日被排為 `statutory_holiday` 或 `day_off` 而自動抵扣。
- `statutory_holiday.used_count` 與 `day_off.used_count` 也分別只計算自身 `day_type`；同一格只計入一種假別，與 Excel 依短名 `國`、`例`、`休` 執行 `COUNTIF` 的口徑一致。
- 若 API 原始資料的補假註解指向另一個週六國定假日，例如 2026-10-09 為 2026-10-10 國慶日補假，則補假日 2026-10-09 納入 `public_holiday` 應排數，原日期 2026-10-10 只作日期欄標記，不納入 `public_holiday` 餘額，也不增加 `day_off.target_count`。
- `public_holiday` 排在一般工作日後仍視為非 `workday`，單格更新與週期確認都必須套用核心班別最小值班人數規則，不得因假別為國定假而略過檢查。
- 例如週期為 2026-09-17 至 2026-10-14 時，2026-10-08 即使不是國假餘額日，只要該人尚有國假額度且更新後符合最小值班人數，仍可排為 `public_holiday`；同一週若尚未排例假，也可排為 `statutory_holiday`。

此檢查：

- 單格儲存後可回傳 warning，但例假超排與最小值班人數不足必須阻擋。
- 週期確認時必須全部通過，否則回傳 409 並列出缺例假、例假超排、休假餘額或國定假餘額異常的人員與區間。

### 國例休餘額

矩陣查詢需為每位工程師計算 `public_holiday`、`statutory_holiday`、`day_off` 三種必要假別的餘額：

```text
target_count    應排數
used_count      已排數
remaining_count 未排餘額，target_count - used_count
```

三種假別的 `used_count` 均以格子 `day_type` 精準計數：`public_holiday` 對應 `國`、`statutory_holiday` 對應 `例`、`day_off` 對應 `休`。`calendar_marks.is_balance_date` 僅參與 `public_holiday.target_count`，不得限制 `國` 的可選日期，也不得讓 `例` 或 `休` 自動增加 `public_holiday.used_count`。

Excel 匯出檢核欄需保留既有班表的「國 / 例 / 休」公式口徑。既有已排數仍以 `public_holiday`、`statutory_holiday`、`day_off` 的 `used_count` 為準；Excel 相容差異在匯出公式，不使用 `remaining_count` 的正負號：

```text
顯示值：
若已排數 > 應排數，顯示「錯」
否則顯示 已排數 - 應排數
```

因此 4 週週期若應排 `國 3`、`例 4`、`休 4`，且某人已排 `國 3`、`例 4`、`休 3`，Excel 匯出檢核欄需顯示 `0 / 0 / -1`。若 `休` 已排 5 天，需顯示 `0 / 0 / 錯`。API 的 `remaining_count` 仍維持 `target_count - used_count`。

前端畫面及會直接呈現在前端的後端驗證訊息採用人類可讀口徑，不顯示負數：

```text
若 used_count < target_count，顯示「缺 target_count - used_count」
若 used_count = target_count，顯示「0」
若 used_count > target_count，顯示「錯」
```

例如上述休假少排 1 天時，前端與確認週期失敗訊息顯示 `休 缺 1`；後端數值欄位與 Excel 匯出公式不因此改變。

2026-09-21 至 2026-10-04 兩週週期範例：

- 週檢查區間：2026-09-21 至 2026-09-27、2026-09-28 至 2026-10-04。
- 每人 `statutory_holiday.target_count = 2`。
- 每人 `day_off.target_count = 2`。
- 若期間特殊國定假日為中秋節與孔子誕辰 / 教師節，則每人 `public_holiday.target_count = 2`。
- 初稿仍全部為 `workday`，所以三者 `remaining_count` 初始皆等於 target。
- `public_holiday.used_count` 只計算排定 `public_holiday` 的格子；特殊國定假日若排為 `statutory_holiday` 或 `day_off`，只計入對應的 `例` 或 `休`。

2026-10-15 至 2026-11-11 四週週期範例：

- 週檢查區間共 4 週。
- 若期間特殊國定假日為 2026-10-25 週日與 2026-10-26 補假，2026-10-25 只作日期標記，不納入國定假餘額；每人 `public_holiday.target_count = 1`。
- 每人 `statutory_holiday.target_count = 4`。
- 每人 `day_off.target_count = 4`。
- 初稿仍全部為 `workday`，所以初始未排餘額應顯示 `國 1`、`例 4`、`休 4`。

2026-10-01 至 2026-10-28 四週週期補假範例：

- 若期間有 2026-10-09 補假，且官方 API raw payload / description 註明其為 2026-10-10 國慶日補假，則 2026-10-09 納入 `public_holiday.target_count`。
- 2026-10-10 仍需在日期欄顯示國慶日與週六欄底色，但不納入 `public_holiday.target_count`，也不得讓 `day_off.target_count` 增加。
- 若工程師把 2026-10-10 排為 `day_off`，只抵扣 `day_off.used_count`，不得抵扣 `public_holiday.used_count`。
- Matrix response 的 `calendar_marks` 需提供前端判斷該日期是否計入國定假餘額的欄位，避免前端仍以「非週日」推算錯誤。
- `calendar_marks.is_balance_date` 只影響 `public_holiday.target_count`；2026-10-10 或週期內其他日期仍可依個人國假額度保存 `day_type = public_holiday`。

2026-09-17 至 2026-10-14 四週週期補假驗收範例：

- 週檢查區間共 4 週。
- 若期間 `calendar_marks` 有 4 筆特殊國定假日標記，其中 1 筆為已有補假日的原始國定假日，該原始日 `is_balance_date` 必須為 `false`。
- 每人 `public_holiday.target_count = 3`、`statutory_holiday.target_count = 4`、`day_off.target_count = 4`。
- 初稿仍全部為 `workday`，所以初始未排餘額應顯示 `國 3`、`例 4`、`休 4`。
- 若原始國定假日被排為 `day_off` 或 `statutory_holiday`，只抵扣對應 `休` 或 `例` 已排數，不抵扣 `國` 已排數。

## 國定假日同步與標記流程

自動國定假日 API 拆成兩個階段：

1. 同步官方資料到 `public_holidays`。
2. 排班矩陣查詢時讀取特殊國定假日，回傳日期標記與每人 `國` 應排餘額。

同步官方資料不得直接修改 `user_schedules`。建立週期與重新整理草稿週期也不得自動把任何格子改為 `public_holiday`。這樣可以避免政府資料更新後影響已確認 / 已鎖定週期，也避免普通週末被誤排為國定假日。

### 官方資料 adapter

第一版支援兩個 provider：

- `data_gov_tw_123662`：主要來源，對應政府資料開放平台 `https://data.gov.tw/dataset/123662`。欄位包含 `date`、`year`、`name`、`isholiday`、`holidaycategory`、`description`。其中 `isholiday` 需轉成內部 `is_holiday`。
- `taiwan_calendar_json`：備援 / 正規化來源，對應 `https://github.com/ruyut/TaiwanCalendar` 與 CDN `https://cdn.jsdelivr.net/gh/ruyut/TaiwanCalendar/data/{year}.json`。欄位包含 `date`、`week`、`isHoliday`、`description`。

預設使用 `data_gov_tw_123662`。`taiwan_calendar_json` 是第三方整理後 JSON，適合做備援或降低 CSV / 編碼處理成本，但資料可信來源仍需標示回政府資料開放平台。

adapter 統一輸出內部模型：

```go
type PublicHolidayCandidate struct {
	Date        time.Time
	Name        string
	IsHoliday   bool
	SourceKey   string
	RawPayload  map[string]any
}
```

adapter 負責：

- 依年度取得官方資料。
- 將民國年、`YYYYMMDD` 字串或外部格式轉為 Asia/Taipei 日期。
- 判斷該筆是否為排班用特殊國定假日；普通週六、週日即使外部資料標示為放假，也不得輸出成 `public_holidays.is_holiday = TRUE`。補班 / 上班日不得輸出成 `public_holiday`。
- 產生穩定 `source_key`，供診斷與差異比較。
- 保留 `raw_payload` 供診斷，但業務邏輯只能使用轉換後欄位。
- 依授權要求保留資料來源標示；response 或審計摘要中需能追溯 provider 與原始 dataset URL。

### 同步模式

`preview`：

- 取得官方資料並與 `public_holidays` 比對。
- 回傳新增、更新、跳過、衝突摘要。
- 不更新 `public_holidays`。

`apply`：

- 執行與 preview 相同的差異比較。
- 新增不存在的 API 資料。
- 更新既有 `source = api` 資料。
- 遇到同日期 `source = manual` 資料時保留 manual，並標示為 skipped。
- 寫入 `public_holiday_sync_runs` 與系統審計。

### 排班矩陣日期標記

建立週期與重新整理草稿時不自動套用國定假日到排班格。矩陣查詢時需提供日期層級標記：

- 讀取週期日期範圍內 `is_holiday = TRUE` 的 `public_holidays`。
- 回傳 `columns.is_weekend`、`calendar_marks` 與每人 `public_holiday` 應排餘額。
- 不更新 `user_schedules`。
- 不寫入 `user_schedule_history`。
- 普通週六、週日以 `columns.is_weekend = true` 標示，不納入 `public_holiday` 應排餘額。
- 週六、週日若同時是特殊國定假日，仍需出現在 `calendar_marks`；是否納入每人 `public_holiday` 應排餘額需依 `calendar_marks.is_balance_date` 判斷。

Matrix response 需額外回傳日期層級標記：

```json
{
  "columns": [
    { "date": "2026-01-03", "weekday": "六", "is_weekend": true }
  ],
  "calendar_marks": [
    { "date": "2026-01-01", "type": "public_holiday", "name": "元旦", "is_balance_date": true }
  ]
}
```

`columns.is_weekend` 只表示週末欄位標示，`calendar_marks` 表示特殊國定假日標記，兩者都不代表該日所有人已排定 `day_type = public_holiday`。前端需以高對比欄底色標示週末與特殊國定假日欄位；若有 `calendar_marks.name`，日期標題需直接顯示名稱，例如「中秋節」。格子仍顯示自身 `day_type`，未手動排國假時仍為班別短名。

## API

### 國定假日

```text
GET    /api/v1/schedule/public-holidays?year=&from=&to=&source=
POST   /api/v1/schedule/public-holidays
PUT    /api/v1/schedule/public-holidays/:id
DELETE /api/v1/schedule/public-holidays/:id
POST   /api/v1/schedule/public-holidays/sync
```

`POST /api/v1/schedule/public-holidays/sync`：

```json
{
  "year": 2026,
  "provider": "data_gov_tw_123662",
  "mode": "preview"
}
```

response：

```json
{
  "year": 2026,
  "provider": "data_gov_tw_123662",
  "mode": "preview",
  "status": "success",
  "summary": {
    "fetched": 18,
    "inserted": 2,
    "updated": 1,
    "skipped": 1
  },
  "changes": [
    { "date": "2026-01-01", "name": "元旦", "action": "insert" },
    { "date": "2026-02-17", "name": "春節", "action": "skipped_manual_override" }
  ]
}
```

### 班別設定

```text
GET    /api/v1/schedule/shift-groups
GET    /api/v1/schedule/duty-shifts
GET    /api/v1/schedule/current-shift-group
POST   /api/v1/schedule/duty-shifts
PUT    /api/v1/schedule/duty-shifts/:id
PATCH  /api/v1/schedule/duty-shifts/:id/enabled
```

`GET /api/v1/schedule/current-shift-group` 回傳目前設定的班制與其啟用班別，供 Ticket 列表、Report 設計器與排班頁面共用：

```json
{
  "shift_group_id": "01...",
  "shift_group_code": "3shift_fixed",
  "shift_group_name": "三班固定制",
  "duty_shifts": [
    { "id": "01...", "code": "morning", "name": "早班", "sort_order": 1 },
    { "id": "01...", "code": "afternoon", "name": "中班", "sort_order": 2 },
    { "id": "01...", "code": "night", "name": "晚班", "sort_order": 3 }
  ]
}
```

### 人員班別歸屬

```text
GET    /api/v1/schedule/shift-assignments?user_id=&date=
POST   /api/v1/schedule/shift-assignments
PUT    /api/v1/schedule/shift-assignments/:id
DELETE /api/v1/schedule/shift-assignments/:id
```

### 排班週期

```text
GET    /api/v1/schedule/periods?status=&from=&to=
POST   /api/v1/schedule/periods
GET    /api/v1/schedule/periods/:id
GET    /api/v1/schedule/periods/:id/matrix
POST   /api/v1/schedule/periods/:id/refresh-draft
PATCH  /api/v1/schedule/periods/:id/schedules/:schedule_id
POST   /api/v1/schedule/periods/:id/confirm
POST   /api/v1/schedule/periods/:id/lock
GET    /api/v1/schedule/periods/:id/export?format=xlsx|csv
```

### 主要 request / response

建立週期：

```json
{
  "name": "20260501-0528",
  "start_date": "2026-05-01",
  "end_date": "2026-05-28",
  "shift_group_id": "01...",
  "labor_law_mode": "4week",
  "remark": ""
}
```

更新單格：

```json
{
  "day_type": "statutory_holiday",
  "remark": "排定例假"
}
```

Matrix response：

```json
{
  "period": {
    "id": "01...",
    "name": "20260501-0528",
    "start_date": "2026-05-01",
    "end_date": "2026-05-28",
    "labor_law_mode": "4week",
    "status": "draft"
  },
  "staffing_policy": {
    "min_staff_per_core_shift": 1,
    "core_shift_codes": ["morning", "afternoon", "night"]
  },
  "columns": [
    { "date": "2026-05-01", "weekday": "五" }
  ],
  "calendar_marks": [
    { "date": "2026-05-01", "type": "public_holiday", "name": "勞動節", "is_balance_date": true }
  ],
  "groups": [
    {
      "duty_shift_id": "01...",
      "duty_shift_name": "早班",
      "users": [
        {
          "user_id": "01...",
          "username": "vincent",
          "display_name": "Vincent",
          "cells": [
            {
              "schedule_id": "01...",
              "date": "2026-05-01",
              "day_type": "workday",
              "editable": true,
              "remark": ""
            }
          ],
          "summary": {
            "workdays": 20,
            "public_holidays": 0,
            "statutory_holidays": 4,
            "day_offs": 4,
            "annual": 0,
            "official": 0,
            "sick": 0,
            "personal": 0,
            "other": 0,
            "weekly_check": {
              "ok": true,
              "missing_weeks": [],
              "over_quota_weeks": []
            },
            "leave_balances": {
              "public_holiday": {
                "target_count": 1,
                "used_count": 0,
                "remaining_count": 1
              },
              "statutory_holiday": {
                "target_count": 4,
                "used_count": 4,
                "remaining_count": 0
              },
              "day_off": {
                "target_count": 4,
                "used_count": 4,
                "remaining_count": 0
              }
            }
          }
        }
      ]
    }
  ],
  "totals": {
    "day_offs": 12,
    "statutory_holidays": 12,
    "public_holidays": 0
  },
  "confirmations": []
}
```

## 權限

- Admin：班別設定、人員班別歸屬管理、鎖定週期。
- SRE 經理：查看、匯出、鎖定週期。
- 值班組長：建立週期、編輯所有班表、確認週期、匯出。
- 工程師：查看週期，編輯自己的 `day_type` 與備註。
- 現行班制查詢：已登入且具備 Ticket / Report / Schedule 任一讀取入口者可讀取；此 endpoint 不回傳敏感設定值，只回傳班制與啟用班別選項。
- 國定假日資料查詢：具備 Schedule 讀取權限者可查詢；系統管理頁入口僅顯示給 `admin` / `op_admin`。
- 國定假日手動維護與官方同步：僅 `admin` / `op_admin` 可操作。
- 國定假日標記：矩陣查詢自動回傳日期標記與餘額，不提供手動或自動改格的週期套用權限。

建議表單權限路徑：

```text
schedule
schedule/shifts
schedule/assignments
schedule/periods
schedule/periods/matrix
system/public-holidays
```

## 匯出

後端負責產生 Excel / CSV。

匯出需包含：

- Sheet 名稱：週期名稱。
- 日期列與星期列。
- 依早 / 中 / 晚分組的人員列。
- 每日排班格。
- 右側每人統計。
- 底部總計。
- 組長簽名欄。

## 與既有需求差異

原 OnCall Ticket System 需求中的三班制描述為輪班。本 spec 明確改為三班固定制。若未來要支援三班輪班，需另開 deferred task，不應混入本階段資料流程。
