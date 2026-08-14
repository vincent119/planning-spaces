# DateField 年份選擇前端設計

## 設計目標

在不更換日期套件、不改各頁日期資料流的前提下，讓共用 `DateField` 支援快速切換年份與月份。

## 現況摘要

`DateField` 目前實作位置：

```text
opscenter-frontend/src/components/DateField.tsx
```

目前結構：

- `TextField` 顯示格式化日期。
- 點擊欄位或月曆 icon 後開啟 `Popover`。
- `Popover` header 有上一月與下一月。
- `monthLabel` 只顯示年月，不可互動。
- 日期格固定顯示 42 天。
- `清除` 與 `今日` 放在底部。

問題點：

- 沒有年份選擇入口。
- 跨年度日期只能逐月切換。
- `TextField` 是唯讀，不能直接輸入年份。

## 元件狀態設計

建議在 `DateField` 內新增 view mode：

```text
day      日期格模式
month    月份選擇模式
year     年份選擇模式
```

狀態建議：

```text
visibleMonth: Date
calendarMode: 'day' | 'month' | 'year'
yearPanelStart: number
```

狀態行為：

- 開啟月曆時固定回到 `day` 模式。
- `visibleMonth` 仍以已選日期或今日初始化。
- 切換年份時只改 `visibleMonth` 年份，保留目前月份。
- 切換月份時只改 `visibleMonth` 月份，保留目前年份。

## Header 設計

### 日期格模式

header 建議呈現：

```text
[上一月] [月份按鈕] [年份按鈕] [下一月]
```

行為：

- 上一月：顯示前一個月。
- 下一月：顯示下一個月。
- 月份按鈕：切到 `month` 模式。
- 年份按鈕：切到 `year` 模式。

### 月份選擇模式

顯示 12 個月份。

行為：

- 點選月份後切回 `day` 模式。
- 保留目前年份。
- 月份名稱使用目前語系。

### 年份選擇模式

顯示一組年份格，建議每頁 12 年。

行為：

- 點選年份後切回 `day` 模式。
- 保留目前月份。
- 年份面板提供上一組 / 下一組，避免硬限制可選年份。

## 可及性設計

需要補齊：

- 開啟月曆 icon 的 `aria-label` 維持既有文案。
- 上一月 / 下一月維持既有 `aria-label`。
- 月份按鈕新增 `calendar.change_month`。
- 年份按鈕新增 `calendar.change_year`。
- 年份面板上一組新增 `calendar.previous_years`。
- 年份面板下一組新增 `calendar.next_years`。
- 月份與年份格使用 button，保留鍵盤 focus。
- 日期、月份、年份格需要有目前選取狀態視覺。
- `TextField` 需提供穩定 `id`，確保 label 與 input 關聯。

## i18n 設計

需更新：

```text
opscenter-frontend/src/locales/zh-TW/common.json
opscenter-frontend/src/locales/zh-CN/common.json
opscenter-frontend/src/locales/en/common.json
```

建議新增 key：

```text
calendar.change_month
calendar.change_year
calendar.previous_years
calendar.next_years
calendar.select_month
calendar.selected_month
calendar.select_year
calendar.selected_year
calendar.months
```

`calendar.months` 可用陣列：

```json
["1月", "2月", "3月", "4月", "5月", "6月", "7月", "8月", "9月", "10月", "11月", "12月"]
```

英文可使用：

```json
["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"]
```

## 視覺設計

保持 `Popover` 寬度接近既有 `304px`。

日期格：

- 維持 7 欄。
- 選取日期維持 `contained`。
- 今日維持 `outlined`。

月份格：

- 建議 3 欄 x 4 列。
- 選取月份使用 `contained`。
- 非選取月份使用 `outlined` 或 `text`，需符合目前主題。

年份格：

- 建議 3 欄 x 4 列。
- 目前年份可使用 `outlined`。
- 目前顯示年份可使用 `contained`。

不得加入過度裝飾色，避免與目前後台主題衝突。

## 影響頁面

共用元件更新後，以下頁面可直接受益：

- `ProjectTicketsPage.tsx`
- `ProjectJiraPage.tsx`
- `ReportCenterPage.tsx`
- `ReportDesignerPage.tsx`
- `ReportTemplateDetailPage.tsx`
- `SchedulePages.tsx`

各頁不應額外建立自己的年份選擇邏輯。

## 風險與處理

### 風險 1：Popover 內容高度變動

處理方式：

- 不在同一畫面同時顯示日期、月份、年份三套格子。
- 使用 mode 切換，避免 Popover 過高。

### 風險 2：使用者切到年份模式後不知道如何返回

處理方式：

- 點選年份後自動回日期格。
- Header 保留目前範圍文字與返回日期格入口。

### 風險 3：i18n 回傳陣列型別不穩定

處理方式：

- 讀取 `calendar.months` 與 `calendar.weekdays` 時需檢查是否為陣列。
- fallback 使用固定月份陣列，避免畫面崩潰。

## 驗證設計

實作後需驗證：

- `npm run typecheck`
- `npm run build`
- 在人員班別頁打開查詢日期，直接選擇不同年份。
- 在排班週期建立畫面打開開始日期與結束日期，直接選擇不同年份。
- 在 Ticket 列表日期篩選打開日期起 / 日期迄，直接選擇不同年份。
- 在 Report 日期設定打開開始日期 / 結束日期，直接選擇不同年份。
