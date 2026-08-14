# Draft: Structured Access Logging

Status: Draft

## Source

- 近期「log 格式有問題，是不是沒套用 zlogger」debug fix

## Intent

用新 SDD 補齊 access log 統一格式的 bugfix 文件，確保 HTTP request log 只走 zlogger 結構化 JSON，不再混入 Gin 預設文字格式。

## Bounded Context

包含：

- HTTP access log middleware
- Logger initialization
- Request id and trace id fields
- Gin default logger disable behavior

不包含：

- 全系統 log schema 重設計
- 外部 log pipeline 部署

## Promotion Notes

Promote 時需在 `tasks.md` 補驗收：啟動服務後呼叫任一 API，stdout 不應再出現 `[GIN] yyyy/mm/dd` 格式。

