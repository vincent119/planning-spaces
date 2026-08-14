# Draft: Ticket Import Tools

Status: Draft

## Source

- `.kiro/specs/2026-06-22_15-24_Import_ticket_tools`

## Intent

重新規劃 Ticket 批次匯入、欄位映射、驗證、dry-run、錯誤回報與匯入結果追蹤。

## Bounded Context

包含：

- 匯入檔格式
- 欄位驗證與錯誤列表
- dry-run 與實際寫入流程
- 匯入權限

不包含：

- 長時間背景 job 平台
- 通用檔案儲存服務

## Promotion Notes

舊 spec 幾乎完成，若只需補文件，可 promote 成 `Docs` 或小範圍 `Feature` 收斂驗收。

