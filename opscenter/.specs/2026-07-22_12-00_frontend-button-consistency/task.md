# 前端按鈕風格一致性任務

## 1. 共用規範與元件

- [x] 1.1 建立共用按鈕樣式來源
  - 新增 `primaryActionButtonSx`
  - 新增 `toolbarActionsSx`
  - 新增 `rowActionsSx`
  - 不再於單頁重複宣告相同漸層樣式
  - _Requirements: 1.1, 2.1, 2.2, 2.5_

- [x] 1.2 建立頁面主操作按鈕元件
  - 支援 `startIcon`
  - 支援 `disabled`
  - 支援 `component={RouterLink}`
  - 視覺對齊既有新增按鈕
  - _Requirements: 1.1, 2.3, 5.1_

- [x] 1.3 建立表格列操作 icon 元件或 helper
  - 統一 `IconButton size="small"`
  - 必填 Tooltip
  - 必填 aria-label
  - 支援 disabled reason
  - _Requirements: 3.1, 3.5, 3.6_

- [x] 1.4 建立工具列按鈕排列規則
  - 搜尋 / 預覽使用 `contained`
  - 刷新 / 匯出 / 同步預覽 / 清空使用 `outlined`
  - 行動版可堆疊
  - _Requirements: 1.2, 1.3, 2.4, 5.4_

## 2. Ticket 與專案工作區按鈕收斂

- [x] 2.1 收斂 Ticket 列表按鈕
  - 抽出 `addButtonSx`
  - 確認搜尋、清空日期、刷新、新增 Ticket 符合共用規範
  - 確認操作欄複製、查看、編輯、刪除符合 icon-only 規範
  - _Requirements: 4.8_

- [x] 2.2 收斂 Webhook 管理按鈕
  - 新增 Webhook 使用共用主按鈕
  - 刷新使用工具操作規則
  - 表格操作欄套用共用 icon 規則
  - _Requirements: 1.1, 1.3, 3.1_

- [x] 2.3 收斂 SLA 管理按鈕
  - 刷新使用工具操作規則
  - 儲存使用共用主操作樣式
  - _Requirements: 1.1, 1.3_

## 3. 排班假勤按鈕收斂

- [x] 3.1 收斂排班週期列表操作
  - 建立週期使用共用主按鈕
  - 查看、CSV、Excel、鎖定、刪除改為 `IconButton + Tooltip`
  - 刪除保留 disabled reason
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 4.2_

- [x] 3.2 收斂排班週期矩陣工具列
  - 重新整理使用 `outlined`
  - CSV / Excel 使用 `outlined`
  - 確認週期 / 鎖定週期使用主操作規則
  - _Requirements: 1.1, 1.3, 4.3_

- [x] 3.3 收斂人員班別頁
  - 新增歸屬使用共用主按鈕
  - 轉班操作套用表格操作欄規則
  - _Requirements: 4.1_

- [x] 3.4 收斂班別設定頁
  - 編輯使用表格操作欄規則
  - 啟用 / 停用使用狀態變更操作規則
  - Dialog 儲存 / 取消依共用規則
  - _Requirements: 4.4_

- [x] 3.5 補齊班別設定新增入口
  - 新增班別使用共用主按鈕
  - 新增 / 編輯共用同一個 Dialog
  - 新增成功後重新載入現行班制
  - _Requirements: 1.1, 4.4_

## 4. 系統管理按鈕收斂

- [x] 4.1 收斂國定假日管理頁
  - 同步預覽使用 `outlined`
  - 套用同步使用主操作規則
  - 新增假日使用共用主按鈕
  - 編輯 / 刪除補齊 Tooltip 與 aria-label
  - _Requirements: 4.5_

- [x] 4.2 收斂全域設定列表
  - 搜尋使用查詢操作規則
  - 新增設定使用共用主按鈕
  - 表格操作欄符合 icon-only 規則
  - _Requirements: 4.6_

- [x] 4.3 收斂全域設定新增 / 編輯頁
  - 儲存使用共用主操作樣式
  - 取消使用低強度按鈕
  - disabled 與錯誤狀態保持不變
  - _Requirements: 4.6_

- [x] 4.4 收斂排程管理頁
  - 新增任務使用共用主按鈕
  - 刷新使用工具操作規則
  - 批次刪除與清除全部使用危險操作規則
  - 任務表格操作欄對齊共用 icon 規則
  - _Requirements: 4.7_

## 5. 報表與其他後台頁補齊

- [x] 5.1 收斂報表中心按鈕
  - 預覽使用查詢操作規則
  - 匯出 CSV 使用工具操作規則
  - 儲存範本 disabled 狀態保留 Tooltip
  - _Requirements: 1.2, 1.3, 4.8_

- [x] 5.2 掃描剩餘重複按鈕樣式
  - 找出剩餘 `addButtonSx`
  - 找出剩餘 `saveButtonSx`
  - 找出剩餘獨立 `searchButtonSx`
  - 決定是否納入本次或建立後續 task
  - _Requirements: 2.1, 2.2_

## 6. i18n 與可及性

- [x] 6.1 補齊 Tooltip i18n
  - 查看
  - 複製
  - 編輯
  - 執行
  - 啟用
  - 停用
  - 匯出
  - 刪除
  - _Requirements: 3.5, 6.5_

- [x] 6.2 補齊 aria-label
  - 所有 icon-only 按鈕都需有可讀 label
  - label 需帶入資料名稱或識別資訊
  - _Requirements: 3.5, 6.5_

- [x] 6.3 補齊 disabled reason 顯示
  - 刪除週期
  - 確認週期
  - 鎖定週期
  - 排程觸發
  - 批次操作
  - _Requirements: 1.6, 3.6_

## 7. 驗證

- [x] 7.1 執行前端型別檢查
  - `npm run typecheck`
  - _Requirements: 6.1_

- [x] 7.2 執行前端 build
  - `npm run build`
  - _Requirements: 6.2_

- [ ] 7.3 深色主題視覺檢查
  - Ticket 列表
  - 排班週期列表
  - 排班週期矩陣
  - 人員班別
  - 國定假日管理
  - 全域設定
  - 排程管理
  - _Requirements: 6.3, 6.4_

- [ ] 7.4 亮色主題抽查
  - 排班週期
  - 全域設定
  - 國定假日管理
  - _Requirements: 5.1, 5.2, 6.4_

- [ ] 7.5 窄螢幕檢查
  - 工具列不互相遮擋
  - 按鈕文字不超出容器
  - 表格操作欄不撐爆版面
  - _Requirements: 5.4_
