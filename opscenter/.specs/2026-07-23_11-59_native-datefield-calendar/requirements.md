# DateField 原生與自製日曆需求

## 文件定位

本 spec 規劃讓共用 `DateField` 支援瀏覽器原生日期輸入，同時保留現有自製月曆作為一致 UI 與回退方案。

本階段只做需求與設計規劃，不實作前端程式碼。

原始設計稿 `../2026-06-01_10-22_oncall-ticket-system` 不在本次修改範圍。

## 背景

目前 `opscenter-frontend/src/components/DateField.tsx` 已支援自製日 / 月 / 年選擇模式，但仍需要維護：

- 日期格、月份格、年份格切換邏輯。
- Popover 定位與鍵盤互動。
- 月份 / 年份 i18n。
- 可及性與瀏覽器差異。

若直接移除自製月曆，會失去跨瀏覽器一致 UI、深色主題完全控制、Popover footer 中的 `清除` / `今日` 操作。

因此本次規劃不刪除自製元件，而是讓 `DateField` 具備雙模式：

- `native`：使用原生 `input[type="date"]`，作為第一階段預設模式，降低年 / 月 / 日選擇邏輯維護成本。
- `custom`：保留目前自製 Popover 月曆，作為一致 UI 與回退方案。

## 範圍

### 本階段包含

- 規劃 native 模式使用 MUI `TextField` + `type="date"`。
- 規劃 `DateField` 支援 `custom` / `native` 雙模式。
- 保留既有自製 Popover、日期格、月份格、年份格。
- 保留 `DateField` 對外 props 契約。
- 保留輸出值為 `YYYY-MM-DD`。
- 規劃清除與今日快捷操作如何呈現。
- 規劃 native 模式驗證後再決定是否擴大套用。
- 規劃影響頁面與驗收方式。

### 本階段不包含

- 不導入 MUI X Date Pickers。
- 不新增日期套件。
- 不修改 API request / response。
- 不改各頁預設日期計算。
- 不處理非 `DateField` 的其他日期欄位。
- 不在第一階段刪除自製月曆。
- 不執行程式碼修改。

## 需求 1：保留自製月曆並新增原生模式

系統需要保留目前自製月曆，並新增可選原生日期輸入模式，避免一次性替換造成 UI 與操作風險。

### 驗收條件

- [ ] 1.1 `DateField` 保留目前自製 `Popover` 月曆模式。
- [ ] 1.2 `DateField` 新增 native 模式，使用 `TextField` 並設定 `type="date"`。
- [ ] 1.3 custom 模式仍支援既有 `day` / `month` / `year` 切換。
- [ ] 1.4 native 模式可透過瀏覽器原生 picker 選擇年份、月份與日期。
- [ ] 1.5 若瀏覽器不支援原生 date picker，native 模式仍可輸入 `YYYY-MM-DD`。
- [ ] 1.6 第一階段未指定模式時使用 native 行為；需要一致 UI 的頁面可指定 custom。

## 需求 2：保留資料契約

系統需要確保日期欄位資料格式不受 UI 替換影響。

### 驗收條件

- [ ] 2.1 `value` 仍為 `YYYY-MM-DD` 或空字串。
- [ ] 2.2 `onChange` 仍只回傳 `YYYY-MM-DD` 或空字串。
- [ ] 2.3 不把瀏覽器顯示格式送到後端。
- [ ] 2.4 不改任何 API 日期參數名稱與格式。

## 需求 3：清除與今日快捷操作

系統需要依不同模式定義清除與今日操作，避免 native 模式限制影響 custom 模式既有體驗。

### 驗收條件

- [ ] 3.1 custom 模式保留目前 Popover 內的 `清除` / `今日` 操作。
- [ ] 3.2 native 模式的 `清除` 不放在 picker 內，改由欄位旁 clear icon 或頁面既有清空按鈕處理。
- [ ] 3.3 native 模式若保留欄位級清除，需使用 icon-only button 並提供 `aria-label`。
- [ ] 3.4 native 模式的 `今日` 快捷不依賴瀏覽器 picker 內部 UI；若保留，需作為欄位外部小按鈕或不納入第一階段。
- [ ] 3.5 今日值仍使用既有 `todayInTaipei()`，避免瀏覽器時區差異影響資料。

## 需求 4：表單與可及性

系統需要維持兩種模式的表單欄位可讀、可操作、可被瀏覽器與輔助工具理解。

### 驗收條件

- [ ] 4.1 `TextField` label 與 input 保持正確關聯。
- [ ] 4.2 `disabled` 時不可編輯或開啟 picker。
- [ ] 4.3 `error` 與 `helperText` 呈現不變。
- [ ] 4.4 欄位可透過鍵盤 focus 與輸入。
- [ ] 4.5 不使用隱藏 input 或覆蓋瀏覽器 picker 的方式破壞可及性。
- [ ] 4.6 custom 模式既有鍵盤與滑鼠操作不得退化。

## 需求 5：視覺與瀏覽器差異

系統需要保留 custom 模式的一致 UI，並接受 native 模式的瀏覽器 UI 差異，但資料契約與欄位外觀仍需保持一致。

### 驗收條件

- [ ] 5.1 欄位本體仍符合目前 MUI `TextField` 深色主題樣式。
- [ ] 5.2 DOM `value` 與送出值固定為 `YYYY-MM-DD`。
- [ ] 5.3 瀏覽器可用自己的 locale 顯示日期，例如斜線格式或本地化格式，但不得影響送出值。
- [ ] 5.4 native 模式需將目前 app locale 傳到實際 HTML input 的 `lang` 屬性。
- [ ] 5.5 Chrome、Edge、Safari 至少需人工驗證。
- [ ] 5.6 原生 picker 彈出層可維持瀏覽器 / 作業系統樣式，不要求完全套用系統深色主題。
- [ ] 5.7 若某瀏覽器只顯示文字輸入，不視為功能失效，但需可輸入合法日期。
- [ ] 5.8 若某頁需要完全一致 UI，可指定或保留 custom 模式。

## 需求 6：影響頁面

共用 `DateField` 更新後，以下頁面需驗證：

- Ticket 列表日期篩選。
- Jira 報表日期篩選。
- Report 日期設定。
- Report 範本詳情日期設定。
- 排班週期建立日期。
- 排班週期列表日期篩選。
- 人員班別查詢日期。
- 人員班別轉班生效日期。

### 驗收條件

- [ ] 6.1 上述頁面不需各自新增日期 picker 邏輯。
- [ ] 6.2 所有頁面仍透過共用 `DateField` 取得 `YYYY-MM-DD`。
- [ ] 6.3 查詢、建立、更新流程的日期參數不變。
