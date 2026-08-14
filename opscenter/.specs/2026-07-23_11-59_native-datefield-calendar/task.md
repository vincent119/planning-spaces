# DateField 原生與自製日曆任務

> 本 task 已核准實作。程式碼已完成元件層改造；跨瀏覽器與 Network request 仍需人工驗收。

## 1. 文件與契約確認

- [x] 1.1 確認 `DateField` 採雙模式設計
  - 保留 custom 自製 Popover 月曆
  - 新增 native 原生日期輸入
  - 不導入 MUI X Date Pickers
  - 不新增日期套件
  - 第一階段不刪除自製月曆
  - _Requirements: 1.1-1.6_

- [x] 1.2 確認 `DateFieldProps` 對外契約不變
  - `value` 仍為 `YYYY-MM-DD` 或空字串
  - `onChange` 仍回傳 `YYYY-MM-DD` 或空字串
  - `disabled` / `error` / `helperText` / `size` / `sx` 保持行為
  - _Requirements: 2.1-2.4, 4.1-4.3_

- [x] 1.3 確認模式切換契約
  - 新增可選 `calendarVariant?: 'custom' | 'native'`
  - 未指定時使用 native 行為
  - 需要一致 UI 的頁面可指定 custom
  - 頁面不需各自實作日期 picker
  - _Requirements: 1.1-1.6, 6.1-6.3_

## 2. 前端元件設計

- [x] 2.1 整理共用 `DateField` 內部結構
  - 保留 custom 實作
  - 新增 native 實作
  - 對外仍由 `DateField` 統一使用
  - 避免重寫 custom 模式
  - _Requirements: 1.1-1.6, 4.6_

- [x] 2.2 實作 native 日期輸入模式
  - 使用 MUI `TextField type="date"`
  - `InputLabel` 固定 shrink
  - 不使用自製 Popover
  - _Requirements: 1.2, 1.4-1.5, 4.1_

- [x] 2.3 日期值正規化
  - 合法 `YYYY-MM-DD` 才回傳
  - 空字串回傳空字串
  - 不把瀏覽器顯示格式送出
  - _Requirements: 2.1-2.4_

- [x] 2.4 清除操作
  - custom 模式維持 Popover 內既有清除
  - native 模式有值時提供清除 icon
  - icon-only button 補 `aria-label`
  - disabled 時不可清除
  - _Requirements: 3.1-3.5, 4.1-4.5_

- [x] 2.5 今日快捷策略
  - custom 模式維持 Popover 內既有今日
  - native 模式第一階段不保留 DateField 內建今日按鈕
  - 若使用者確認需要，另開 `showTodayShortcut` prop
  - 今日值需使用 `todayInTaipei()`
  - _Requirements: 3.1, 3.4-3.5_

## 3. i18n 與樣式

- [x] 3.1 補或確認 `calendar.clear_date`
  - `zh-TW`
  - `zh-CN`
  - `en`
  - _Requirements: 3.2, 4.1_

- [x] 3.2 保留既有 calendar 舊 key
  - custom 模式仍需要月份、年份、週文字 key
  - 第一階段不刪除既有 calendar key
  - 避免語系檔 churn
  - _Requirements: 1.1, 5.1-5.7_

- [x] 3.3 確認深色主題外觀
  - custom 模式維持既有一致外觀
  - native 模式不覆蓋瀏覽器 picker 內部 UI
  - native 模式只維持 MUI TextField 外層一致
  - _Requirements: 5.1-5.8_

- [x] 3.4 設定 native input 語系提示
  - 將 app locale 傳給實際 HTML input 的 `lang` 屬性
  - 不使用 i18next 文案硬改原生 picker
  - _Requirements: 5.3-5.8_

## 4. 影響頁面驗收

- [ ] 4.1 Ticket / Jira / Report 日期欄位
  - Ticket 列表日期篩選
  - Jira 報表日期篩選明確使用 custom 模式，避免原生 picker 無法完整套用 i18n 或不顯示 Popover
  - Report 日期設定
  - Report 範本詳情日期設定
  - _Requirements: 6.1-6.3_

- [ ] 4.2 Schedule 日期欄位
  - 排班週期建立日期
  - 排班週期列表日期篩選
  - 人員班別查詢日期
  - 人員班別轉班生效日期
  - _Requirements: 6.1-6.3_

- [ ] 4.3 跨瀏覽器手動驗證
  - Chrome
  - Edge
  - Safari
  - custom / native 兩種模式至少各驗一次
  - _Requirements: 5.3-5.8_

## 5. 驗證

- [x] 5.1 執行前端型別檢查
  - `npm run typecheck`
  - _Requirements: 1.1-6.3_

- [x] 5.2 執行前端 build
  - `npm run build`
  - _Requirements: 1.1-6.3_

- [x] 5.3 檢查格式差異
  - `git diff --check`
  - _Requirements: 1.1-6.3_

- [ ] 5.4 Network request 驗證
  - 日期參數仍為 `YYYY-MM-DD`
  - 清空時不送出非法日期
  - _Requirements: 2.1-2.4_
