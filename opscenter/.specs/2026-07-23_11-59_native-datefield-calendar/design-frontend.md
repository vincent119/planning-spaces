# DateField 原生與自製日曆前端設計

## 設計目標

讓共用 `DateField` 支援瀏覽器原生日期輸入，同時保留目前自製月曆，降低替換風險並保留跨瀏覽器一致 UI 的回退方案。

目標不是建立另一套完整日期元件，而是在既有 `DateField` 上增加 native 模式，讓頁面可以依需求選擇「低維護成本」或「一致 UI」。

## 現況

目前 `DateField` 位於：

```text
opscenter-frontend/src/components/DateField.tsx
```

目前行為：

- 使用 MUI `TextField` 顯示格式化日期。
- 點擊欄位開啟自製 `Popover`。
- 自製月曆支援日期、月份、年份切換。
- `onChange` 回傳 `YYYY-MM-DD`。
- `清除` 與 `今日` 放在 Popover 底部。

## 目標架構

建議將目前邏輯拆成兩個內部實作，但對外仍由 `DateField` 統一匯出：

```text
DateField
├── NativeDateInput
└── CustomDateInput
```

對外新增可選模式：

```ts
calendarVariant?: 'custom' | 'native'
```

第一階段預設值：

```text
calendarVariant = 'native'
```

原因：

- 需求目標是讓現有日期欄位優先使用原生日期輸入。
- 自製月曆仍完整保留，若某頁需要一致 UI，可指定 `calendarVariant="custom"`。
- 若 Safari、深色主題、操作體驗不符合預期，可逐頁切回 custom 模式。

native 模式核心：

```tsx
<TextField
  type="date"
  value={value}
  onChange={(event) => onChange(normalizeNativeDateValue(event.target.value))}
/>
```

保留 `DateFieldProps`：

```text
disabled?: boolean
error?: boolean
helperText?: string
label: string
onChange: (value: string) => void
calendarVariant?: 'custom' | 'native'
size?: 'small' | 'medium'
sx?: SxProps<Theme>
value: string
```

不新增頁面層日期邏輯。

## 自製模式保留策略

目前自製 Popover 月曆需要保留，原因是它提供原生 picker 做不到或做不穩的能力：

- 跨瀏覽器一致的日 / 月 / 年選擇體驗。
- 完整配合深色主題。
- Popover footer 內的 `清除` / `今日`。
- 既有 i18n 文案與鍵盤 / 滑鼠互動。

實作階段應避免刪除自製月曆邏輯。若需要整理檔案，建議只做內部命名拆分，例如：

```text
DateField.tsx
DateFieldNative.tsx
DateFieldCustom.tsx
```

是否拆檔由實作時依現有程式碼複雜度決定；不應為了新增 native 模式而重寫 custom 模式。

## 資料格式

兩種模式都必須維持相同資料契約。

HTML `input[type="date"]` 的 DOM `value` 是 `YYYY-MM-DD`。

設計規則：

- `DateField.value` 只接受 `YYYY-MM-DD` 或空字串。
- `event.target.value` 若符合 `YYYY-MM-DD`，直接回傳。
- 若使用者清空，回傳空字串。
- 若瀏覽器允許輸入但值不合法，回傳空字串或維持原值需在實作前二選一。

建議採用：

```text
合法日期 -> onChange(YYYY-MM-DD)
空字串 -> onChange('')
不合法日期 -> 不呼叫 onChange，並讓瀏覽器 validity 處理
```

原因：

- 避免把半輸入狀態送進查詢條件。
- 保留原生 input 的鍵盤輸入能力。

## Native 模式 Label 與顯示

原生 date input 的 DOM `value` 固定是 `YYYY-MM-DD`，但畫面上的顯示格式由瀏覽器與使用者環境決定。

本系統應接受：

- Chrome 可能依系統 locale 顯示日期。
- Safari 可能顯示不同格式或文字輸入。
- 實際送出值仍固定為 `YYYY-MM-DD`。
- native 模式需把 app locale 傳到實際 HTML input 的 `lang` 屬性，讓瀏覽器有正確語系提示。
- 不以畫面顯示格式判斷資料是否正確，驗收以 DOM value 與 Network request 為準。

`InputLabel` 建議固定 shrink：

```tsx
slotProps={{
  htmlInput: { lang: locale },
  inputLabel: { shrink: true },
}}
```

原因：

- `type="date"` 在部分瀏覽器空值時 label 容易與 placeholder / date UI 重疊。
- 原生 picker 不讀 i18next 的月份 / 星期文案，只能透過 `lang` 提示瀏覽器語系。

## 清除與今日

custom 模式保留目前 Popover footer 內的 `清除` / `今日`。

native 模式的原生 picker 內部不能加入自訂 footer，因此需要調整。

### 清除

native 模式建議第一階段保留欄位級清除：

- 有值時顯示 clear icon。
- 點擊後 `onChange('')`。
- icon-only button 必須有 `aria-label="清除日期"`。
- disabled 時不顯示或不可點。

### 今日

native 模式建議第一階段不在 `DateField` 內提供今日按鈕。

理由：

- 原生 date picker UI 由瀏覽器控制，無法在 picker 內放今日按鈕。
- 若在欄位旁加今日按鈕，會讓篩選列變擠。
- 多數使用場景已有日期區間預設或清空日期按鈕。

若產品仍需要今日快捷，建議後續新增可選 prop：

```text
showTodayShortcut?: boolean
```

使用者點擊時呼叫：

```text
onChange(todayInTaipei())
```

## Native 模式開啟 picker

原生 date input 可以由使用者點擊欄位或瀏覽器內建 calendar icon 開啟。

可選擇性使用 `HTMLInputElement.showPicker()`，但不建議第一階段依賴。

原因：

- `showPicker()` 瀏覽器支援度不完全一致。
- 使用原生元件的重點是降低瀏覽器差異維護，而不是再包一層自訂開啟邏輯。

## i18n

因為 custom 模式會保留，自製月份、年份、上一組年份等月曆文案也需要保留。

native 模式若保留 clear icon，需保留或新增：

```text
calendar.clear_date
```

若保留 clear icon 使用。

custom 模式仍需要：

- `calendar.months`
- `calendar.weekdays`
- `calendar.previous_month`
- `calendar.next_month`
- `calendar.change_month`
- `calendar.change_year`
- `calendar.previous_years`
- `calendar.next_years`
- `calendar.select_month`
- `calendar.selected_month`
- `calendar.select_year`
- `calendar.selected_year`

第一階段不得刪除上述 key，避免 custom 模式或其他元件被影響。

## 視覺設計

兩種模式都需維持既有 MUI `TextField` 外觀：

- `size` 仍由 props 控制。
- `sx` 原樣傳入。
- `disabled`、`error`、`helperText` 保持不變。
- 欄位寬度由使用頁既有 layout 控制。

native 模式若加入 clear icon：

- 使用 MUI icon button。
- 保持 32px 以內觸控區域。
- 不覆蓋瀏覽器原生 date picker icon。

## 風險

### 風險 1：瀏覽器顯示格式由環境決定

說明：

- 這不是資料格式風險。HTML `input[type="date"]` 的 DOM `value` 仍固定為 `YYYY-MM-DD`。
- 差異只在畫面呈現，例如 Chrome、Edge、Safari 可能依作業系統 locale 顯示不同格式。

處理：

- 驗收重點放在 DOM value 與 Network request 均為 `YYYY-MM-DD`。
- 不要求所有瀏覽器顯示完全相同的日期文字。
- 需要一致顯示時，使用或保留 custom 模式。

### 風險 2：Safari 原生 date picker 行為與 Chrome 不同

處理：

- Safari 必須人工驗證。
- 若無 picker，至少要能輸入合法日期。
- Safari 體驗不符合需求的頁面，使用或保留 custom 模式。

### 風險 3：清除 / 今日行為改變

處理：

- custom 模式不改變既有 `清除` / `今日`。
- native 模式清除保留在欄位級 icon。
- native 模式今日快捷暫不保留，若使用者確認需要再新增可選 prop。

### 風險 4：原生 picker 彈出層不能完全客製

說明：

- 欄位本體仍是 MUI `TextField`，可配合目前深色主題。
- 受瀏覽器控制的是點開後的原生日期 picker 彈出層，不是整個 input 欄位。

處理：

- 只控制 `TextField` 外層樣式。
- 可設定 `color-scheme` 輔助瀏覽器選擇深色原生 UI，但不把原生 picker 視為可完全客製的元件。
- 不嘗試深度覆蓋瀏覽器 picker UI。
- 需要完全配合深色主題時，使用或保留 custom 模式。

## 驗證

自動驗證：

```bash
cd opscenter-frontend
npm run typecheck
npm run build
git diff --check
```

手動驗證：

- Chrome：Ticket 列表日期起迄、Report 日期起迄。
- Edge：排班週期建立日期、人員班別查詢日期。
- Safari：至少驗證輸入、清除、送出值。

資料驗證：

- Network request 中日期參數仍為 `YYYY-MM-DD`。
- 清空欄位時不送出非法日期。
