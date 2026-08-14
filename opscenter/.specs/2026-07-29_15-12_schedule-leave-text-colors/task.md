# 排班週期假別文字色 Tasks

Status: Complete

## 文件定位

本文件追蹤 `requirements.md` 與 `design.md` 的實作工作。既有排班色彩 task `7.11` 已完成，本次以新的未完成 task 管理三種假別的主題文字色需求。

## Execution Context

### 意圖

讓排班週期表內「國／例／休」在 `dark`、`dark-glass`、`light`、`light-glass` 四種主題使用使用者指定的黃、藍、綠文字色。

### 非目標

- 不改其他假別、出勤格、統計 Chip、下拉選單或日期欄。
- 不改全域 MUI palette。
- 不改深色按鈕背景、其他假別背景、尺寸或排班互動。
- 不改後端與資料 contract。

### 已定決策

- `dark` 的「國／例／休」使用 `#FFD54A`／`#BFE3FF`／`#83FFD5`。
- `dark-glass` 的「國／例／休」使用 `#FFE082`／`#D6ECFF`／`#A8FFE4`。
- `light` 的「國／例／休」使用 `#B45309`／`#1D4ED8`／`#047857`。
- `light-glass` 的「國／例／休」使用 `#9A3412`／`#1E40AF`／`#065F46`。
- 四種主題共用同一個 Component Token，依完整 `ThemeMode` 選擇色碼。
- disabled 與唯讀狀態維持目前主題的完整指定色，不降低透明度。
- 淺色與淺色毛玻璃的「國／例／休」提高背景與邊框透明度，其他假別及深色主題保留既有邏輯。
- 淺色一般背景／邊框使用 `24%`／`82%`，hover 背景使用 `32%`，disabled 背景／邊框使用 `16%`／`58%`。
- 配額 Chip 採 tonal 風格：國紅、例藍、休青；每週缺少維持 warning，超排維持 error，正確維持 success outlined。

### 邊界

修改限於排班 feature 內的 token、排班格樣式與本規格文件。若需要修改全域 theme、其他元件或後端，必須先更新 task Boundary 並取得使用者安排。

### 關鍵檔案

- `opscenter-frontend/src/features/schedule/styles/tokens.ts`
- `opscenter-frontend/src/features/schedule/pages/SchedulePages.tsx`

### 完成條件

- 四組 ThemeMode 色階在一般與 disabled 狀態符合需求。
- 其他假別與工作日沒有樣式回歸。
- `npm run typecheck`、`npm run build`、`git diff --check` 通過。

## Protected Behavior

- 深色排班格與其他假別的背景、邊框繼續依既有主題邏輯計算。
- 其他假別與出勤班別維持既有文字色。
- 唯讀格仍可辨識，但不得恢復可操作狀態。
- 不新增全域 theme 欄位；四組指定色碼集中在排班 Component Token。
- 不修改使用者既有的其他工作區變更。
- 配額、每週檢查的數值、標籤、Tooltip、欄位位置與排列方式不得變更。

## 1. 元件 token 與樣式實作

- [x] 1.1 建立排班假別文字 Component Token
  - Boundary:
    - Allowed Changes: `src/features/schedule/styles/tokens.ts`。
    - Forbidden: 全域 theme、其他 feature token 與後端。
  - Depends: 無。
  - Context: 以 Material primitive 建立 `day_off`、`public_holiday`、`statutory_holiday` 的 light／dark Component Token，提供型別安全查詢。
  - Verify: 執行 `npm run typecheck`。

- [x] 1.2 套用三種假別的一般與 disabled 文字色
  - Boundary:
    - Allowed Changes: `src/features/schedule/pages/SchedulePages.tsx`。
    - Forbidden: 背景、邊框、尺寸、互動、其他假別與工作日樣式重構。
  - Depends: 1.1。
  - Context: 依 `palette.mode` 選擇完整 token；disabled 不使用 `alpha()` 改寫三個指定色，其他假別維持既有透明度。
  - Verify: 執行 `npm run typecheck` 與 `npm run build`。

## 2. 驗證

- [x] 2.1 執行自動驗證與差異檢查
  - Boundary:
    - Allowed Changes: 僅修正本需求直接造成的型別或 build 問題。
    - Forbidden: 刪除檢查、放寬 TypeScript 設定或修改無關檔案。
  - Depends: 1.2。
  - Context: 驗證 production build、型別與差異範圍。
  - Verify:
    - `npm run typecheck`
    - `npm run build`
    - `git diff --check`
    - `git diff --stat`

- [x] 2.2 四主題人工視覺驗收
  - Boundary:
    - Allowed Changes: 僅記錄四主題與一般、唯讀狀態的驗收結果。
    - Forbidden: 未經使用者確認自行調整指定色碼。
  - Depends: 2.1。
  - Context: 在排班週期表切換四種 ThemeMode，使用瀏覽器 computed style 或等價方式確認實際文字色。
  - Verify:
    - 暗色毛玻璃：休 `#B388FF`、國 `#FF80AB`、例 `#64FFDA`。
    - 深色：休 `#B388FF`、國 `#FF80AB`、例 `#64FFDA`。
    - 淺色：休 `#512DA8`、國 `#C2185B`、例 `#00796B`。
    - 淺色毛玻璃：休 `#512DA8`、國 `#C2185B`、例 `#00796B`。
    - confirmed / locked 的 disabled 格子仍使用目前主題的完整色碼。

## 3. 四主題文字色覆寫

- [x] 3.1 將排班文字 Component Token 改為四種 ThemeMode
  - Boundary:
    - Allowed Changes: `src/features/schedule/styles/tokens.ts`。
    - Forbidden: 全域 theme、背景、邊框、其他 feature 與後端。
  - Depends: 已完成 task 1.1。
  - Context: 以 `ThemeMode` 直接保存使用者指定的黃、藍、綠四主題色碼，覆寫第一版 light／dark 兩組 token。
  - Verify: 執行 `npm run typecheck`。

- [x] 3.2 讓排班格依目前 ThemeMode 選取文字色
  - Boundary:
    - Allowed Changes: `src/features/schedule/pages/SchedulePages.tsx`。
    - Forbidden: 背景、邊框、尺寸、互動、其他假別與工作日樣式重構。
  - Depends: 3.1。
  - Context: 從既有 App Context 取得 `themeMode` 並傳入 `scheduleCellButtonSx`；三種指定假別的一般與 disabled 狀態使用完整色碼。
  - Verify: 執行 `npm run typecheck` 與 `npm run build`。

- [x] 3.3 加強淺色三種指定假別的按鈕色彩
  - Boundary:
    - Allowed Changes: `src/features/schedule/pages/SchedulePages.tsx` 與本規格文件。
    - Forbidden: 深色透明度、其他假別、工作日、尺寸、互動、全域 theme 與後端。
  - Depends: 3.2。
  - Context: 淺色 disabled 背景原為 `10%`，在白色系背景不易辨識；只對存在四主題文字 Component Token 的「國／例／休」提高一般、hover 與 disabled 背景／邊框透明度。
  - Verify: `npm run typecheck`、`npm run build`、`git diff --check`，並檢視使用者提供的淺色畫面。

## 4. 本次變更驗證

- [x] 4.1 執行自動驗證與差異檢查
  - Boundary:
    - Allowed Changes: 僅修正本次覆寫直接造成的型別或 build 問題。
    - Forbidden: 放寬 TypeScript 設定或修改無關檔案。
  - Depends: 3.3。
  - Context: 驗證四主題 token、production build 與差異範圍。
  - Verify: `npm run typecheck`、`npm run build`、`git diff --check`、`git diff --stat`。

- [x] 4.2 使用者預覽四主題文字色
  - Boundary:
    - Allowed Changes: 僅記錄使用者預覽結果。
    - Forbidden: 未經使用者確認調整指定色碼。
  - Depends: 4.1。
  - Context: 使用者要求先套用後查看畫面；在四主題確認「國／例／休」及 disabled 狀態。
  - Verify: 對照 requirements 驗收條件 3.1 至 3.5。

## 5. 配額 Chip tonal 風格

- [x] 5.1 建立配額 Chip Component Token 與樣式 helper
  - Boundary:
    - Allowed Changes: `src/features/schedule/styles/tokens.ts`、`src/features/schedule/pages/SchedulePages.tsx`。
    - Forbidden: 全域 theme、資料型別、後端、配額與每週檢查計算。
  - Depends: 3.1。
  - Context: 將國紅、例藍、休青以及 warning／error／success 狀態集中為排班元件樣式；深色使用背景／邊框 `16%`／`60%`，淺色使用 `12%`／`48%`。
  - Verify: `npm run typecheck`。

- [x] 5.2 套用配額與每週檢查 Chip tonal 外觀
  - Boundary:
    - Allowed Changes: `src/features/schedule/pages/SchedulePages.tsx`。
    - Forbidden: Chip 文字、Tooltip、欄位、排列、配額數值與每週檢查邏輯。
  - Depends: 5.1。
  - Context: 高度 `24px`、`1px` 邊框、pill 圓角、字重 `600`、無陰影；資訊元件不增加 hover 浮起效果。
  - Verify: `npm run typecheck`、`npm run build`。

- [x] 5.3 驗證四主題與三種狀態
  - Boundary:
    - Allowed Changes: 僅修正本 task 造成的樣式、型別或 build 問題並回填驗收紀錄。
    - Forbidden: 未經使用者確認變更指定色系、透明度或資料邏輯。
  - Depends: 5.2。
  - Context: 檢查缺額、超排、正確在四種主題的可讀性，以及 Tooltip 與現有排列沒有回歸。
  - Verify: `npm run typecheck`、`npm run build`、`git diff --check` 與四主題人工驗收。

## 品質檢查清單

### 第一版紀錄

- [x] 三組 light／dark 色階只在 Component Token 定義一次。
- [x] 四種主題只依 `palette.mode` 映射，未各自重複定義色碼。
- [x] 同一主題下，一般與 disabled 狀態使用相同指定色。
- [x] 背景與邊框未被改動。
- [x] 其他假別與工作日樣式未被改動。
- [x] 全域 theme 與後端未被改動。
- [x] `npm run typecheck` 通過。
- [x] `npm run build` 通過。
- [x] `git diff --check` 通過且差異未超出 Boundary。

### 本次四主題覆寫

- [x] 四組色碼只在排班 Component Token 定義一次。
- [x] 排班格以完整 `ThemeMode` 索引文字色。
- [x] 同一主題下一般與 disabled 狀態使用相同指定色。
- [x] 深色背景、其他假別、工作日、全域 theme 與後端未被改動。
- [x] 淺色「國／例／休」的一般、hover、disabled 背景與邊框使用指定透明度。
- [x] `npm run typecheck` 通過。
- [x] `npm run build` 通過。
- [x] `git diff --check` 通過且差異未超出 Boundary。
- [x] 使用者已預覽並接受四種主題畫面。

### 配額 Chip tonal 風格

- [x] 國／例／休使用各自 category tone，不再全部呈現實心橘色。
- [x] 每週缺少、超排、正確仍保留 warning／error／success 語意。
- [x] 深色與淺色透明度符合 Component Token。
- [x] 高度、邊框、圓角、字重與陰影符合設計。
- [x] 計算、文字、Tooltip、欄位與排列沒有回歸。
- [x] `npm run typecheck`、`npm run build`、`git diff --check` 通過。
- [x] 使用者已預覽四主題與缺額、超排、正確狀態。

## Implementation Notes

- 2026-07-29：建立新規格，覆寫既有已完成 task `7.11` 的三種文字色行為，不修改舊 task 完成狀態。
- 2026-07-29：依 `ui-design-tokens` 採 feature 內 Component Token，四種主題共用同一 token 結構，並依 `palette.mode` 選擇 light／dark 色階。
- 2026-07-29：使用者接受 Material 色階建議；淺色主題改用 700，深色主題改用 A100／A200，保留紫、粉紅、藍綠三個語意色相。
- 2026-07-29：使用者已安排執行，task 狀態切換為 `InProgress`。
- 2026-07-29：完成 Component Token 與排班格文字色套用；`npm run typecheck`、`npm run build`、`git diff --check` 均通過。
- 2026-07-29：本次執行環境沒有可連接的瀏覽器，task `2.2` 四主題人工視覺驗收尚未執行，因此維持 `InProgress`。
- 2026-07-29：使用者已實際檢視四種主題畫面並確認結果，task `2.2` 驗收完成，規格狀態更新為 `Complete`。
- 2026-07-29：使用者要求先套用黃、藍、綠四主題獨立色碼供預覽；新增 task `3.1` 至 `4.2`，規格狀態切換為 `InProgress`。
- 2026-07-29：完成 task `3.1`、`3.2`、`4.1`；`npm run typecheck`、`npm run build`、`git diff --check` 均通過，等待使用者預覽 task `4.2`。
- 2026-07-29：依使用者要求先直接加強淺色按鈕，完成後補入 task `3.3`；一般背景／邊框調整為 `24%`／`82%`，hover 背景為 `32%`，disabled 背景／邊框為 `16%`／`58%`。使用者提供淺色畫面確認三種按鈕已可辨識。
- 2026-07-29：新增配額 Chip tonal 風格需求與 task `5.1` 至 `5.3`；本次只更新文件，尚未安排實作。
- 2026-07-29：使用者安排執行 task `5.1` 至 `5.3`；完成 Component Token、配額與每週檢查 tonal 樣式及自動驗證，task `5.3` 保留四主題人工預覽。
- 2026-07-29：使用者確認 task `5.3` 四主題與缺額、超排、正確三種狀態驗收通過；配額 Chip tonal 風格工作全部完成。
- 2026-07-29：使用者確認 task `4.2` 四主題文字色驗收通過；所有 task 與驗收條件完成，規格狀態更新為 `Complete`。
