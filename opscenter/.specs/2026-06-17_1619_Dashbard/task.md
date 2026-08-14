# Dashboard Backend Tasks

## 0. Contract

- [x] 0.1 對齊現有 Dashboard API 與 OpenAPI
  - 檢查 `internal/dashboard` 目前 response
  - 檢查 `Docs/openapi.json` 是否包含 `/dashboard` 與 `/projects/{id}/dashboard`
  - 建立缺口清單，不得假設前端 fallback 等於 API 已完成
  - 完成：確認 `internal/dashboard` 目前已有 `/dashboard` 與 `/projects/:id/dashboard` handler，但舊 response 只有 flat summary，不符合 Dashboard snapshot contract
  - 完成：確認 `Docs/openapi.json` 目前未產出 Dashboard paths，需由後續 `1.3` 補 swagger 註解並重新產生 OpenAPI
  - 缺口：trend、priority_distribution、recent_tickets 尚未由 repository 查詢產生，保留給 `2.2`、`2.3`、`2.4`

- [x] 0.2 建立 Dashboard snapshot DTO
  - `scope`
  - `metrics`
  - `trend`
  - `priority_distribution`
  - `recent_tickets`
  - `generated_at`
  - 完成：已新增 Dashboard domain snapshot model 與 delivery response DTO
  - 完成：API response 已由 flat summary 改為 snapshot shape；SLA breach 先回 `source=not_implemented`，圖表與最近 Ticket 先回空陣列
  - 驗證：`go test ./internal/dashboard`

## 1. API

- [x] 1.1 補強 global Dashboard API
  - `GET /api/v1/dashboard`
  - 支援 timezone query
  - admin / op_admin 全部專案
  - member 只看有 project_members 的專案
  - 完成：`timezone` query 會傳入 service，預設 `Asia/Taipei`，無效時區回 400
  - 完成：沿用 repository 既有 admin / op_admin 全域查詢與 member 專案成員範圍查詢

- [x] 1.2 補強 project Dashboard API
  - `GET /api/v1/projects/{id}/dashboard`
  - viewer 權限檢查
  - 回傳 project scope metadata
  - 完成：project Dashboard 透過 project service 驗證 viewer 權限並取得 project metadata
  - 完成：response `scope` 已回傳 `project_id`、`project_name`、`timezone`

- [x] 1.3 補 Swagger / OpenAPI
  - delivery godoc
  - nested response schemas
  - 執行 `make openapi`
  - 完成：已補 `GET /api/v1/dashboard` 與 `GET /api/v1/projects/{id}/dashboard` godoc
  - 完成：已執行 `make openapi`，`Docs/openapi.json` 產出 Dashboard paths 與 nested response schema
  - 驗證：`go test ./internal/dashboard`
  - 驗證：`go test ./...`

## 2. Repository

- [x] 2.1 實作 metrics 聚合
  - open_tickets
  - created_today
  - resolved_today
  - pending_p1_p2
  - sla_breaches 回傳 0 / not_implemented
  - 完成：保留既有 `Summary(ctx, QueryInput)` 聚合查詢，並由 service 層將 `sla_breaches` 固定為 `0` 且 `source=not_implemented`

- [x] 2.2 實作 7 日 trend
  - created count
  - resolved count
  - timezone day bucket
  - 完成：新增 repository trend 查詢，依 `QueryInput.Timezone` 以 PostgreSQL day bucket 聚合 created / resolved；service 補齊 7 日零值

- [x] 2.3 實作 priority distribution
  - P1/P2/P3/P4
  - 未結束 Ticket
  - 完成：新增 repository priority distribution 查詢，範圍限定未結束 Ticket；service 固定回傳 P1/P2/P3/P4

- [x] 2.4 實作 recent tickets
  - 最近 10 筆
  - 權限範圍一致
  - 完成：新增 repository recent tickets 查詢，依建立時間倒序取 10 筆，並 left join project / sub_project 名稱；權限範圍與 Summary 共用

## 3. Test

- [x] 3.1 補 service 權限測試
  - global admin
  - global member
  - project viewer
  - project permission denied
  - 完成：已補 global admin / member actor scope 傳入 repository 的驗證
  - 完成：已補 project viewer 使用 `GetProject` / `RequireProjectAccess` 且 required role 為 viewer 的驗證
  - 完成：已補 project 權限不足時不執行 repository 的驗證

- [x] 3.2 補 repository SQL 測試或 integration test
  - metrics
  - trend
  - priority distribution
  - recent tickets
  - 完成：已補 metrics admin / member / project scope、trend member / project scope、priority 未結束條件與 admin / project scope、recent tickets project scope 與 limit 10 SQL 驗證

- [x] 3.3 補 delivery handler test
  - 401
  - 400 invalid timezone
  - 403
  - 404
  - 200 snapshot
  - 完成：已補 401、invalid timezone 400、project 403、project 404，並保留 snapshot 200 response contract 驗證
  - 驗證：`go test ./internal/dashboard`
