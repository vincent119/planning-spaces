# Draft: Ticket Workflow

Status: Draft

## Source

- `.kiro/specs/2026-06-15_15-35_Ticket`

## Intent

重新定義 Ticket 建立、查詢、編輯、狀態流轉、詳情頁、附件、活動紀錄與前後端協作流程。

## Bounded Context

包含：

- Ticket CRUD 與列表查詢
- Ticket 狀態與欄位 contract
- 專案、子專案、成員、資源的讀取依賴
- 前端 Ticket 頁面的資料載入順序

不包含：

- SLA checker
- 通知發送
- 報表聚合
- 分區維護排程

## Promotion Notes

Promote 前需先確認 Ticket 權限來源，避免 Dashboard、SLA、Report 各自查 membership 造成讀取規則不一致。

