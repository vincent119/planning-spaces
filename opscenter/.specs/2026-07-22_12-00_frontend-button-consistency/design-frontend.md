# 前端按鈕風格一致性設計

## 設計目標

本設計把後台按鈕拆成固定語意層級，讓使用者在不同功能頁看到相同樣式時，可以推斷相同操作風險與重要性。

## 現況摘要

目前 theme 只集中設定 `Paper`、`Card`、`Dialog`、`Menu`、`Popover` 等容器樣式，尚未集中設定 `MuiButton`。

頁面內已發現下列重複或不一致：

- `ProjectTicketsPage.tsx`、多個 admin / ticket 子頁使用重複的 `addButtonSx`。
- `AdminSystemEditPage.tsx`、`ProfilePage.tsx` 使用重複或相近的 `saveButtonSx`。
- `AdminLogsPage.tsx` 另有獨立 `searchButtonSx`。
- `SchedulePages.tsx` 的排班週期列表與矩陣工具列使用未指定 `variant` 的文字按鈕。
- `AdminPublicHolidaysPage.tsx` 的同步預覽是文字按鈕，套用同步與新增假日是一般 `contained`。
- `AdminSchedulersPage.tsx` 的任務操作欄使用 `IconButton + Tooltip`，但排班週期列表操作欄使用文字按鈕。

## 共用元件規劃

建議新增共用 UI 元件或共用樣式檔，位置可放在：

```text
opscenter-frontend/src/components/actions/
```

建議元件：

```text
PagePrimaryActionButton.tsx
ToolbarActionButton.tsx
RowActionIconButton.tsx
DialogActionButtons.tsx
```

若不想一次拆多個元件，也可先建立：

```text
opscenter-frontend/src/components/actions/buttonStyles.ts
```

集中提供：

```text
primaryActionButtonSx
toolbarActionsSx
rowActionsSx
```

## 按鈕層級

### 頁面主操作

適用操作：

- 新增 Ticket
- 新增使用者
- 新增設定
- 新增任務
- 建立週期
- 新增歸屬
- 新增班別
- 新增假日

建議規則：

- 使用 `variant="contained"`。
- 使用 `startIcon`，例如 `AddIcon`、`SaveIcon`。
- 使用共用主按鈕樣式。
- 位於頁面標題列右側或篩選列右側時，行動版需可堆疊。

若保留既有漸層視覺，需集中定義，不得在頁面重複宣告。

### 查詢操作

適用操作：

- 搜尋
- 預覽
- 報表預覽

建議規則：

- 使用 `variant="contained"`。
- 使用 `SearchIcon` 或語意相符 icon。
- 不使用新增按鈕的漸層樣式，避免與建立資料混淆。

### 工具操作

適用操作：

- 刷新
- 重新整理
- CSV
- Excel
- 匯出 CSV
- 同步預覽
- 清空日期

建議規則：

- 使用 `variant="outlined"`。
- 使用 `RefreshIcon`、`DownloadIcon`、`SyncIcon` 等 icon。
- 排列在查詢操作或主操作旁邊時，需保持同高與同間距。
- 不使用純文字按鈕。

### 狀態變更操作

適用操作：

- 確認週期
- 鎖定週期
- 啟用
- 停用
- 觸發排程

建議規則：

- 大型流程主狀態變更可使用 `contained`。
- 表格單列狀態變更使用 `IconButton + Tooltip`。
- 啟用可用 `success`；停用可用 `warning`。
- 需保留 disabled reason。

### 危險操作

適用操作：

- 刪除
- 批次刪除
- 清除全部

建議規則：

- 使用 `color="error"`。
- 表格單列刪除使用 `IconButton + DeleteIcon + Tooltip`。
- 批次刪除或 Dialog 確認刪除可使用 `Button variant="contained" color="error"`。
- 清除全部若不是刪除資料但會大量移除紀錄，仍需使用警示色或危險色，並保留確認流程。

## 表格操作欄設計

DataGrid / table 操作欄統一規則：

```text
查看 -> 複製 -> 編輯 -> 執行 / 啟停 -> 匯出 -> 刪除
```

呈現方式：

- 使用 `Stack direction="row"`。
- 使用 `IconButton size="small"`。
- 每個按鈕包 `Tooltip`。
- 每個按鈕有 `aria-label`。
- 停用按鈕外層加 `<span>`，確保 Tooltip 可顯示。
- 欄寬依 icon 數量設定，不以文字長度撐大。

排班週期列表目前的文字操作按鈕需改為此規則。

## 頁面工具列設計

### 篩選區

篩選區操作建議順序：

```text
搜尋 -> 清空 / 清除日期 -> 刷新 -> 新增
```

若「新增」是頁面主操作，優先放在頁面標題列右側；若頁面沒有標題列工具區，才放在篩選列尾端。

### 明細頁工具列

明細頁操作建議順序：

```text
刷新 / 重新整理 -> 匯出 -> 狀態變更主操作
```

例如排班週期矩陣：

```text
重新整理 outlined
CSV outlined
Excel outlined
確認週期 contained
鎖定週期 contained
```

## 優先頁面調整設計

### Ticket 列表

Ticket 列表目前接近目標規則，可作為參考基準：

- 搜尋：`contained`
- 清空日期：`outlined`
- 刷新：`outlined`
- 新增 Ticket：主按鈕
- 操作欄：複製、查看、編輯、刪除使用 icon-only

需補齊：

- 抽出 `addButtonSx`。
- 確認刪除 disabled / 權限狀態與 Tooltip。

### 排班週期列表

需調整：

- 建立週期改用共用主按鈕。
- 查看、CSV、Excel、鎖定、刪除改為 `IconButton + Tooltip`。
- 刪除停用原因保留 Tooltip。

### 排班週期矩陣

需調整：

- 重新整理改為 `outlined`。
- CSV / Excel 改為 `outlined`。
- 確認週期、鎖定週期維持 `contained`，但套用共用主操作規則。
- disabled 狀態需在畫面上可理解。

### 人員班別

需調整：

- 新增歸屬改用共用主按鈕。
- 轉班若在表格列內，改為 `IconButton + Tooltip`；若保留文字，需有一致 row action 樣式並控制欄寬。

### 班別設定

需調整：

- 新增班別放在頁面標題列右側，使用共用主按鈕樣式。
- 編輯、啟用 / 停用改為表格操作欄共用 icon 規則。
- 新增 / 編輯共用同一個 Dialog，儲存 / 取消依共用規則。

### 國定假日管理

需調整：

- 同步預覽改為 `outlined`。
- 套用同步維持 `contained` 或主操作樣式。
- 新增假日改用共用主按鈕。
- 表格編輯 / 刪除維持 icon-only，但補齊 aria-label 與 disabled reason。

### 全域設定

需調整：

- 新增設定使用共用主按鈕。
- 搜尋使用查詢操作規則。
- 編輯設定頁儲存使用共用主操作樣式。
- 取消使用低強度按鈕，需與其他 Dialog / Form 一致。

### 排程管理

需調整：

- 新增任務使用共用主按鈕。
- 刷新、批次刪除、清除全部維持工具 / 危險操作規則。
- 任務表格目前已接近 `IconButton + Tooltip`，需作為操作欄基準並補齊一致 spacing。

### 報表中心

需調整：

- 預覽使用查詢操作規則。
- 匯出 CSV 使用工具操作規則。
- 儲存範本若尚未開放，disabled 呈現需一致且 Tooltip 說明原因。

### Webhook 管理

需調整：

- 新增 Webhook 使用共用主按鈕。
- 刷新使用工具操作規則。
- 表格操作欄套用 icon-only 規則。

### SLA 管理

需調整：

- 刷新使用工具操作規則。
- 儲存使用共用主操作樣式。

## i18n 與可及性

- 所有 Tooltip 文字使用繁體中文 i18n key。
- 不得在 component 內新增不可翻譯硬編碼英文訊息。
- icon-only 按鈕必須有 `aria-label`。
- disabled reason 若需要顯示，需使用現有 i18n namespace 或新增對應 namespace key。

## 驗證策略

本階段實作完成後需執行：

```bash
npm run typecheck
npm run build
```

視覺驗證：

- 深色主題：Ticket 列表、排班週期、人員班別、國定假日管理、全域設定、排程管理。
- 亮色主題：至少抽查排班週期與全域設定。
- 窄螢幕：確認工具列按鈕可換行，不遮擋內容。

## 風險與注意事項

- 抽共用元件時不得改變既有 onClick、權限、disabled 條件。
- 不得因統一表格操作欄而移除既有 Tooltip 停用原因。
- 不得把排班週期的刪除、鎖定等權限行為改掉；本 spec 只規範呈現。
- 不得讓 `IconButton` 缺少可讀 label，否則鍵盤與讀屏操作會退化。
