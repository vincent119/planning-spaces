# DateField 年份選擇任務

> 已核准執行：2026-07-23 使用者指示 `go`。

## 1. 文件與契約確認

- [x] 1.1 確認共用 `DateField` 為唯一修改入口
  - 不在各頁新增重複年份選擇邏輯
  - 不更換日期套件
  - _Requirements: 1.1-1.5, 3.1-3.6_

- [x] 1.2 確認影響頁面清單
  - Ticket 列表
  - Jira 報表
  - Report 日期設定
  - 排班週期
  - 人員班別
  - _Requirements: 5.3-5.6_

## 2. DateField UI 行為

- [x] 2.1 新增月曆模式狀態
  - `day`
  - `month`
  - `year`
  - 開啟月曆時預設回到 `day`
  - _Requirements: 1.1-1.5_

- [x] 2.2 調整月曆 header
  - 保留上一月 / 下一月
  - 月份文字改為可切換月份模式
  - 年份文字改為可切換年份模式
  - _Requirements: 1.1-2.4_

- [x] 2.3 新增月份選擇模式
  - 顯示 12 個月份
  - 點選月份後回到日期格
  - 依目前語系顯示月份
  - _Requirements: 2.2-2.4_

- [x] 2.4 新增年份選擇模式
  - 顯示 12 年一組
  - 支援上一組 / 下一組年份
  - 點選年份後回到日期格
  - 不硬限制只能選固定年份區間
  - _Requirements: 1.1-1.5_

## 3. i18n 與可及性

- [x] 3.1 補齊 `common.calendar` 文案
  - `zh-TW`
  - `zh-CN`
  - `en`
  - _Requirements: 4.1-4.5_

- [x] 3.2 補齊月份 / 年份控制的 `aria-label`
  - 月份切換
  - 年份切換
  - 上一組年份
  - 下一組年份
  - 月份格
  - 年份格
  - _Requirements: 4.1-4.4_

- [x] 3.3 確認 `TextField` label 與 input 關聯
  - 補穩定 `id`
  - 避免瀏覽器可及性檢查出現 label 未關聯警告
  - _Requirements: 4.6_

## 4. 既有行為保護

- [x] 4.1 保留日期選取輸出格式
  - 點選日期仍回傳 `YYYY-MM-DD`
  - _Requirements: 3.1_

- [x] 4.2 保留清除與今日行為
  - `清除` 回傳空字串
  - `今日` 使用台北時區今日
  - _Requirements: 3.2, 3.3_

- [x] 4.3 保留 disabled / error / helperText 行為
  - disabled 不可開啟月曆
  - error 與 helperText 呈現不變
  - _Requirements: 3.4, 3.5_

## 5. 驗證

- [x] 5.1 執行前端型別檢查
  - `npm run typecheck`
  - _Requirements: 5.1_

- [x] 5.2 執行前端 build
  - `npm run build`
  - _Requirements: 5.2_

- [ ] 5.3 手動驗證人員班別日期
  - 查詢日期可直接選年份
  - 程式覆蓋：人員班別使用共用 `DateField`
  - _Requirements: 5.3_

- [ ] 5.4 手動驗證排班週期日期
  - 建立週期開始日期 / 結束日期可直接選年份
  - 程式覆蓋：排班週期使用共用 `DateField`
  - _Requirements: 5.4_

- [ ] 5.5 手動驗證 Ticket 與 Report 日期
  - Ticket 日期篩選可直接選年份
  - Report 日期設定可直接選年份
  - 程式覆蓋：Ticket、Jira、Report 使用共用 `DateField`
  - _Requirements: 5.5, 5.6_

- [x] 5.6 檢查格式差異
  - `git diff --check`
  - _Requirements: 1.1-5.6_
