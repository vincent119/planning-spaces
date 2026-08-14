# 任務文件：Policy Hot Reload 與 Generation 一致性

Status: Complete

## Execution Context

- 意圖：讓已啟動的 vault-agent 定期載入有效 policy，並以程序內 atomic generation 同時供 webhook 與 sync worker 使用。
- 非目標：不改 policy schema、rule matching、Vault/AWS policy、Kubernetes API RBAC、cluster-wide barrier 或 in-flight request cancellation。
- 已定決策：polling；預設 30 秒；0 停用；穩定 snapshot；atomic pointer；last-known-good；相同內容不增加 generation；安全 metrics/log。
- 邊界：config、domain snapshot/reloadable authorizer、application reloader、main lifecycle、metrics、sample/deployment config、繁中文件與本 spec。
- 關鍵檔案：`internal/configs`、`internal/syncer/domain/policy.go`、`internal/syncer/application`、`cmd/vault-agent/main.go`、`internal/infra/metrics/metrics.go`
- 完成條件：所有驗收情境、race test、vet、lint、CLI validation、Kustomize render、文件與 diff checks 完成。

### Protected Behavior

- `SecretAuthorizer` 介面、policy v1 schema、rule matching 與 deny-by-default 不變。
- 初始 policy 無效時 startup 繼續 fail-closed。
- `policy_file` / `policy_dir` 互斥與 `policy validate` CLI contract 不變。
- Readiness 不因 background reload failure 失敗；last-known-good 繼續服務。
- Kubernetes RBAC、Dockerfile、`/app/policy` directory mount 與 webhook failure policy 不變。
- Runtime error/log 保持英文且不輸出 policy content、resource path、keys、fingerprint 或 credential。
- `_workspace/` 不得修改或納入提交。

### 邊界

#### Allowed Changes

- `internal/configs/config.go`
- `internal/configs/config_test.go`
- `internal/syncer/domain/policy.go`
- `internal/syncer/domain/policy_test.go`
- `internal/syncer/application/policy_reloader.go`
- `internal/syncer/application/policy_reloader_test.go`
- `internal/syncer/application/mutate_usecase_test.go`
- `internal/syncer/application/sync_worker_usecase_test.go`
- `internal/syncer/delivery/webhook_handler_test.go`
- `cmd/vault-agent/main.go`
- `cmd/vault-agent/main_test.go`
- `internal/infra/metrics/metrics.go`
- 必要的既有 metrics tests 或新測試檔
- `configs/config.sample.yaml`
- `deployments/kustomize/base/deployment.yaml`
- `docs/config.zh-Hant.md`
- `docs/deploy.zh-Hant.md`
- `.specs/2026-08-04-17-18_Feature-policy-hot-reload/`

#### Forbidden

- 修改 policy YAML public schema 或 authorization matching
- 新增 fsnotify、Kubernetes informer 或其他 dependency
- 新增 Kubernetes API 權限、Role、ClusterRole 或 ServiceAccount permission
- 將 ConfigMap 改為 `subPath` mount
- 新增管理 HTTP endpoint 或 signal handler
- 修改 Vault/AWS server-side policy 或 client permission
- 記錄 policy resource values、完整 YAML、fingerprint 或 credential
- 修改 `_workspace/`

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 建立 config 與 snapshot red tests | 無 | Complete | 已確認缺少新 contract 的 compile failure |
| T2 實作一致 PolicySnapshot loader | T1 | Complete | 重用既有 strict decode/validate |
| T3 建立 atomic reloadable authorizer | T1、T2 | Complete | 保持 SecretAuthorizer 介面 |
| T4 實作 PolicyReloader state machine | T2、T3 | Complete | fake tick、last-known-good |
| T5 串接 main lifecycle 與 shared authorizer | T3、T4 | Complete | startup、shutdown、disabled path |
| T6 新增 metrics 與安全日誌 | T4、T5 | Complete | 固定低基數 label |
| T7 更新設定、部署與操作文件 | T5、T6 | Complete | 未新增 RBAC |
| V1 驗收與回歸驗證 | T1 至 T7 | Complete | 全套品質 gate 通過 |

## 實作任務

- [x] T1 建立 config 與 snapshot red tests
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/configs/config_test.go`、`internal/syncer/domain/policy_test.go`
    - Forbidden: production code、main、metrics、deployment 與文件
  - Depends: 無
  - Context: 固定 interval 預設 30、0 停用、負數拒絕；file/dir deterministic fingerprint；來源變更、空目錄、有效空規則與敏感 marker contracts。
  - Verify:
    - `go test ./internal/configs ./internal/syncer/domain` 應先因缺少新 contract 而失敗
    - `git diff --check`

- [x] T2 實作一致 PolicySnapshot loader
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/configs/config.go`、`internal/syncer/domain/policy.go`、對應 tests
    - Forbidden: 修改 YAML schema、matching、CLI flags、第三方 dependency
  - Depends: T1
  - Context: 從排序後 source bytes 建立 PolicySnapshot；使用長度前綴的 basename/content 計算 SHA-256；保留 symlink escape、strict YAML、單一 document、版本與 duplicate ID 驗證；`LoadPolicy` 維持相容 wrapper。
  - Verify:
    - `go test ./internal/configs ./internal/syncer/domain`
    - `go test ./cmd/vault-agent -run 'TestRunCLI'`
    - fingerprint 不出現在 formatter 或 error output

- [x] T3 建立 atomic reloadable authorizer
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/domain/policy.go`、`internal/syncer/domain/policy_test.go`
    - Forbidden: application use case 內加入 reload 判斷、原地修改 Policy
  - Depends: T1、T2
  - Context: immutable generation state 由 `atomic.Pointer` 保存；每個 Authorize 方法只 Load 一次；publish 只接受有效且不同 fingerprint 的 snapshot；generation 1 起算。
  - Verify:
    - `go test -race ./internal/syncer/domain -run 'TestReloadable|TestPolicy'`
    - 高並行測試只能觀察完整舊或新 decision set

- [x] T4 實作 PolicyReloader state machine
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/application/policy_reloader.go`、`internal/syncer/application/policy_reloader_test.go`
    - Forbidden: 真實 sleep 測試、server startup、readiness、HTTP endpoint
  - Depends: T2、T3
  - Context: interval tick 後載入候選並重取 fingerprint；source changed、invalid、unchanged、success 分流；錯誤保留 last-known-good；context cancel 立即停止。
  - Verify:
    - `go test -race ./internal/syncer/application -run 'TestPolicyReloader'`
    - tests 使用可控制 tick/source，不依賴 wall-clock timing

- [x] T5 串接 main lifecycle 與 shared authorizer
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/main.go`、`cmd/vault-agent/main_test.go`
    - Forbidden: 修改 HTTP handler contract、readiness、external client initialization、引入 errgroup
  - Depends: T3、T4
  - Context: startup 同步載入 generation 1；webhook 與 sync worker 注入同一 reloadable instance；interval 0 不啟動 runner；shutdown 取消並等待 reloader，沿用既有 lifecycle pattern。
  - Verify:
    - `go test -race ./cmd/vault-agent -run 'Test(Build|Start|Wait|Policy)'`
    - startup invalid policy、disabled authorization 與 zero interval regressions

- [x] T6 新增 metrics 與安全日誌
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/infra/metrics/metrics.go`、必要 metrics tests、T4/T5 對應檔案
    - Forbidden: 動態或高基數 labels、raw loader error、policy path/content/fingerprint 輸出
  - Depends: T4、T5
  - Context: generation gauge、reload result counter、last success Unix time；固定 success/unchanged/failure；log 只含 generation 與固定 error category。
  - Verify:
    - `go test ./internal/infra/metrics ./internal/syncer/application ./cmd/vault-agent`
    - 敏感 marker 不出現在 captured log 或 metric labels

- [x] T7 更新設定、部署與操作文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: `configs/config.sample.yaml`、`deployments/kustomize/base/deployment.yaml`、`docs/config.zh-Hant.md`、`docs/deploy.zh-Hant.md`
    - Forbidden: RBAC、Dockerfile、volume `subPath`、英文使用文件
  - Depends: T5、T6
  - Context: 說明 30 秒預設、0 停用、ConfigMap propagation＋interval 延遲、last-known-good 風險、有效空規則 deny-all、空目錄 failure、per-Pod generation skew 與監控方式。
  - Verify:
    - `rg -n 'reload_interval_seconds|generation|last-known-good|空規則|空目錄' configs deployments docs`
    - `kubectl kustomize deployments/kustomize/base`
    - `kubectl kustomize deployments/kustomize/overlays/prod`
    - render 結果仍是 `/app/policy` directory mount 且沒有 `subPath`

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Verify: requirements 的 valid、invalid、source changed、unchanged、deny-all、disabled、shared generation 與安全觀測情境均有 automated tests。

- [x] V1 回歸驗證
  - Verify: `go test -race -count=1 ./...`、`go vet ./...`、`make lint`、`make policy-validate`、base/prod Kustomize render。

- [x] V1 品質檢查清單
  - [x] config default、0 與負數 contract 通過
  - [x] startup 初始 policy invalid 仍 fail-closed
  - [x] valid candidate atomic publish 通過
  - [x] invalid、空目錄與 source changed 保留 last-known-good
  - [x] 有效空規則 policy 發布後 deny all
  - [x] unchanged content 不增加 generation
  - [x] webhook 與 sync worker 共用 authorizer
  - [x] concurrent authorization race test 通過
  - [x] reloader cancellation 與 wait 通過
  - [x] metrics labels 保持固定低基數
  - [x] log/error 不含敏感 marker 或 fingerprint
  - [x] CLI policy validation 回歸通過
  - [x] Go race test、vet 與 lint 通過
  - [x] base/prod Kustomize render 通過
  - [x] Kubernetes RBAC 未變更且 volume 未使用 `subPath`
  - [x] 文件一致性已確認
  - [x] `git diff --stat` 已檢查
  - [x] `git diff --check` 已通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的 `Execution Context`
2. 目前未完成 task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個 `.specs` 目錄。若文件很大，先用標題與關鍵字定位：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-17-18_Feature-policy-hot-reload
```

## Implementation Notes

- 2026-08-04：已完成 discovery 與正式 spec，尚未修改 production code、tests、config、deployment 或 user-facing docs。
- 2026-08-04：使用者再次確認 Go 測試失敗訊息也必須使用英文；任務邊界加入既有中文測試訊息所在檔案，統一修正但不改測試邏輯。
- 2026-08-04：T1 已新增 config default/zero/negative 與 snapshot contract tests，並確認 production contract 尚未存在時的 compile failure。
- 2026-08-04：T2/T3 已完成 bytes-based 穩定 snapshot、SHA-256 內容識別、前後來源一致性檢查、immutable generation 與 atomic publish；既有 `LoadPolicy` 與 CLI 維持相容。
- 2026-08-04：T4 已新增可注入 ticker/loader/clock 的 `PolicyReloader`，valid、unchanged、invalid、source changed、deny-all 與 cancellation tests 通過。
- 2026-08-04：T5 已讓 webhook 與 sync worker 共用 reloadable authorizer，正 interval 啟動 reloader，0 維持 startup-only，初始無效 policy 繼續 fail-closed，並納入 graceful wait。
- 2026-08-04：T6 已新增 generation、固定三種 reload result 與最後成功時間 metrics；背景錯誤只記錄固定英文摘要，不輸出 raw policy error 或 fingerprint。
- 2026-08-04：T7 已更新 sample config、base Deployment env 與繁中維運文件；RBAC 未變更，`/app/policy` 維持 directory mount 且未使用 `subPath`。
- 2026-08-04：V1 `go test -race -count=1 ./...`、`go vet ./...`、`make policy-validate`、`go list ./...`、base/prod Kustomize render 與 diff checks 均通過。
- 2026-08-04：本機預設 `golangci-lint v2.12.1` 被版本 gate 拒絕；依 `.golangci-lint-version` 將 v2.12.2 安裝至 `/private/tmp` 後執行 `make lint`，結果為 `0 issues`，未修改或繞過專案固定版本。
- 既有 `PolicyAuthorizer` 保存固定 `*Policy`；webhook 與 sync worker 已透過同一 `SecretAuthorizer` contract 使用授權，因此可由 reloadable wrapper 取代而不修改 use case 行為。
- 既有 deployment 已採 `/app/policy` directory mount 且未使用 `subPath`，符合 Kubernetes ConfigMap volume 更新前提。
- Polling 刻意取代 fsnotify/Kubernetes API watch，以避免 projected volume symlink 事件問題與新增 RBAC。
- Background failure 採 last-known-good；這代表無效撤銷不會生效，必須由 CI validation、metrics 與告警共同控管。
- Generation number 是 process-local counter，不代表 cluster-wide policy identity。
- `_workspace/` 不屬於本 spec，不納入任何變更。

## 驗證結果摘要

- 新行為驗證：通過；stable snapshot、atomic generation、last-known-good、deny-all、unchanged、shared authorizer 與 cancellation 均有測試
- 回歸驗證：通過；全套 race test、vet、固定版本 lint、policy validation、go list 與 Kustomize render 成功
- 文件一致性：requirements、design、tasks、sample config、Deployment 與繁中操作文件已對齊
- 剩餘風險：多 Pod 仍可能因 ConfigMap propagation 與 polling interval 短暫使用不同 generation，此限制已納入 metrics 與文件

## 後續改善

- [ ] 另案規劃跨 replica 的 rollout observation 或 generation identity。
- [ ] 另案評估手動 reload trigger；須先定義認證、授權與審計契約。
- [ ] 最終階段執行 Kubernetes failure/recovery 演練。
