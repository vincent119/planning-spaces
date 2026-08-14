# 排班週期假別文字色設計

## 文件定位

本設計實作 `requirements.md` 定義的排班週期「休／國／例」主題文字色，接續既有排班矩陣 `scheduleCellButtonSx` 樣式。

本設計只覆寫三種 day type 的文字色，不重寫已完成的排班矩陣色彩系統。

## 已知契約狀態

### 需求來源

- 使用者要求以指定黃、藍、綠色碼覆寫第一版紫、粉紅、藍綠方案，並先套用四主題供預覽。
- `requirements.md` 本次驗收條件 3.1 至 3.7。

### Theme contract

- `ThemeMode` 包含 `light`、`light-glass`、`dark`、`dark-glass`。
- 四種主題共用 `SchedulePages.tsx` 的 `scheduleCellButtonSx`。
- `theme.palette.mode` 只能區分 `light` 與 `dark`，不足以選擇毛玻璃專用色碼。
- `useAppContext()` 可提供完整 `ThemeMode`，由 `ScheduleMatrixPage` 傳入 `scheduleCellButtonSx`。

### Component contract

- `dayTypeShort` 將 `day_off`、`public_holiday`、`statutory_holiday` 顯示為「休」、「國」、「例」。
- `leaveDayTypeTone` 與 `leaveToneColor` 提供假別背景、邊框與一般文字 tone。
- `scheduleCellButtonSx` 的 `&.Mui-disabled` 目前會使用 `alpha(tone.text, ...)` 降低假別文字透明度。
- 不可假造新的 MUI palette 欄位，除非同時補 Theme 型別擴充；本需求不需要擴充全域 Theme。

## Bounded Context

### 包含

- 排班矩陣按鈕內三種假別文字色。
- 可編輯、hover、disabled 與唯讀狀態。
- 四種 ThemeMode 的人工視覺驗收。
- 淺色與淺色毛玻璃中「國／例／休」按鈕的背景與邊框透明度。

### 不包含

- 其他排班頁、統計 Chip、下拉選單與日期欄。
- 全域 palette 與其他功能元件。
- 深色與暗色毛玻璃按鈕背景、其他假別背景、尺寸或互動。
- 後端、API、資料庫與排班驗證規則。

## 設計原則

依 `ui-design-tokens` 採用排班元件層 token：

1. 色碼只在一個 token 檔案定義。
2. 元件依 `ScheduleDayType` 的語意取得文字色。
3. 四個主題共用相同 token，不建立重複 theme override。
4. 四種 ThemeMode 直接對應四組 token，不擴充全域 MUI Theme。
5. 背景與邊框繼續使用既有 tone 與 `alpha()`。
6. 指定文字色優先於既有 disabled 透明度規則。
7. 淺色的背景強化只在三種指定假別有 Component Token 時生效，不影響其他假別。

## Token 設計

新增排班元件 token：

```ts
type ScheduleLeaveTextColors = Partial<Record<ScheduleDayType, Record<ThemeMode, string>>>;

export const scheduleLeaveTextColors: ScheduleLeaveTextColors = {
  public_holiday: {
    dark: '#FFD54A',
    'dark-glass': '#FFE082',
    light: '#B45309',
    'light-glass': '#9A3412',
  },
  statutory_holiday: {
    dark: '#BFE3FF',
    'dark-glass': '#D6ECFF',
    light: '#1D4ED8',
    'light-glass': '#1E40AF',
  },
  day_off: {
    dark: '#83FFD5',
    'dark-glass': '#A8FFE4',
    light: '#047857',
    'light-glass': '#065F46',
  },
};
```

建議位置：

```text
opscenter-frontend/src/features/schedule/styles/tokens.ts
```

此 token 是排班 Component Token，集中保存使用者指定色碼，不放入全域 MUI palette，避免 Ticket、Report 或其他功能誤用排班專用語意色。

## 樣式選擇流程

```text
有排班資料
  ├─ workday：維持既有班別 palette
  └─ leave day type
      ├─ day_off：依 ThemeMode 取得綠色 token
      ├─ public_holiday：依 ThemeMode 取得黃色 token
      ├─ statutory_holiday：依 ThemeMode 取得藍色 token
      └─ 其他假別：文字維持 leaveToneColor(theme, tone).text
```

`scheduleCellButtonSx` 仍以 `leaveToneColor()` 的 `main` 計算背景與邊框，只將 `color` 改為：

```ts
const themedTextColor = scheduleLeaveTextColors[dayType]?.[themeMode];
const textColor = themedTextColor ?? tone.text;
```

實作時需使用型別安全的 helper 或 type guard，避免以任意字串索引 token。

## 狀態規則

| 狀態 | 三種指定假別文字色 |
|---|---|
| 一般 | 使用目前 `ThemeMode` 的完整 token 色碼 |
| hover | 使用目前 `ThemeMode` 的完整 token 色碼 |
| disabled | 使用目前 `ThemeMode` 的完整 token 色碼，不套 `alpha()` |
| confirmed / locked | 按鈕雖 disabled，仍使用目前主題的完整 token 色碼 |

其他假別的 disabled 文字仍沿用既有透明度計算。

### 淺色按鈕狀態透明度

| 狀態 | 原值 | 「國／例／休」新值 |
|---|---:|---:|
| 一般背景 | `16%` | `24%` |
| 一般邊框 | `72%` | `82%` |
| hover 背景 | `24%` | `32%` |
| disabled 背景 | `10%` | `16%` |
| disabled 邊框 | `42%` | `58%` |

`dark` 與 `dark-glass` 保持既有一般背景 `30%`、邊框 `96%`、hover 背景 `42%`、disabled 背景 `20%`、disabled 邊框 `58%`。其他假別在淺色主題也保持原值。

## 配額 Chip tonal 設計

### 設計目的

目前 `balanceChipColor()` 將所有缺額映射為 MUI `warning` filled，因此國缺、例缺、休缺與每週缺少皆呈現高飽和實心橘色。新設計保留文字中的「缺／超排／正確」狀態，不改計算邏輯，並讓前三種配額 Chip 沿用排班格的國紅、例藍、休青色系。

### Component Token

| Chip | 深色背景／邊框基礎色 | 深色文字 | 淺色背景／邊框基礎色 | 淺色文字 |
|---|---|---|---|---|
| 國缺 `public_holiday` | `#F87171` | 依 `ThemeMode` 使用國文字 token | `#DC2626` | 依 `ThemeMode` 使用國文字 token |
| 例缺 `statutory_holiday` | `#60A5FA` | 依 `ThemeMode` 使用例文字 token | `#2563EB` | 依 `ThemeMode` 使用例文字 token |
| 休缺 `day_off` | `#22D3EE` | 依 `ThemeMode` 使用休文字 token | `#0891B2` | 依 `ThemeMode` 使用休文字 token |
| 每週缺少 | MUI warning main | warning contrast text | MUI warning main | warning dark text |

背景與邊框透明度：

| 主題群組 | 背景 | 邊框 |
|---|---:|---:|
| `dark`／`dark-glass` | `16%` | `60%` |
| `light`／`light-glass` | `12%` | `48%` |

### 狀態規則

- 缺額：前三種使用各自 category tonal 色；每週缺少使用 warning tonal 色。
- 超排：使用 error tonal 色，提高至背景 `18%`、邊框 `65%`。
- 正確：維持 success outlined，背景透明，降低正常狀態的視覺權重。
- Chip 文字已包含「缺」、「超排」或正確狀態，不以顏色作為唯一資訊來源。

### 外觀規格

- 高度 `24px`。
- 邊框 `1px`。
- 圓角使用 pill。
- 字重 `600`。
- 不加陰影。
- Chip 為資訊元件，不加入 hover 浮起、pressed 或指標游標。
- 保留目前 `size="small"` 的資訊密度、Tooltip 與排列方式。

## 受影響檔案計畫

- `opscenter-frontend/src/features/schedule/styles/tokens.ts`
  - 新增排班元件文字色 token 與型別安全查詢。
- `opscenter-frontend/src/features/schedule/pages/SchedulePages.tsx`
  - 三種假別套用 token，disabled 狀態保持指定色，並加強淺色背景與邊框透明度。
  - 套用配額與每週檢查 Chip 的 tonal 狀態樣式，不改計算與 Tooltip。
- `.kiro/specs/2026-07-29_15-12_schedule-leave-text-colors/task.md`
  - 追蹤實作、驗證與人工主題檢查。

## 關鍵行為

- 毛玻璃與非毛玻璃主題依完整 `ThemeMode` 使用各自文字色。
- `ScheduleMatrixPage` 負責把 App Context 的 `themeMode` 傳給排班格樣式。
- disabled selector 不得以透明度改寫目前主題對應的三個文字色。
- 其他假別仍可依 `palette.mode` 使用不同文字 tone。
- 淺色與淺色毛玻璃只加強「國／例／休」背景與邊框；深色及其他假別維持既有值。

## 風險與處理方式

### 不同背景的可讀性

四種主題使用使用者指定的獨立色碼；其中毛玻璃背景可能因漸層與底圖產生局部差異，因此保留使用者四主題預覽驗收。若需再次變更色碼，應新增 task 追蹤覆寫原因與結果。

### MUI disabled 樣式覆寫

MUI disabled 狀態可能套用預設文字色。現有 `&.Mui-disabled` 已有元件層覆寫，實作需在該 selector 明確設定固定 token，確保 computed color 正確。

### token 範圍擴散

排班專用色不應放入全域 palette。使用 feature 內的 Component Token，降低其他頁面意外套用風險。

## 驗證設計

- TypeScript 編譯確認 token key 與 `ScheduleDayType` 對齊。
- production build 確認樣式可打包。
- 四主題各檢查一般與 disabled 排班格 computed color。
- 抽查其他假別與工作日，確認未套用三個固定 token。
- `git diff --check` 確認格式與邊界。
