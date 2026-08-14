# 任務文件：Sync Update Conflict Retry

Status: Complete

## Execution Context

- 意圖: 讓 Sync Worker 對 Kubernetes Secret `409 Conflict` 執行 bounded、context-aware retry，同時保留最新 metadata 並防止 reference 改變後的 stale data write。
- 非目標: 不重試 fetch/list/非 conflict error；不新增 config、dependency、metrics、RBAC、deployment 或 server-side apply。
- 已定決策: 第一次 update 不 GET；conflict 後 GET 最新物件；重驗 opt-in 與 parsed reference；只替換 Data；使用 `retry.DefaultBackoff` 參數搭配 `wait.ExponentialBackoffWithContext`；reference 改變時交由下一輪重新授權/fetch。
- 邊界: 變更限於 Sync Worker application 實作/測試、必要的 K8s repository error contract、README 與本 spec。
- 關鍵檔案: `internal/syncer/application/sync_worker_usecase.go`、`internal/syncer/application/sync_worker_usecase_test.go`、`internal/syncer/infra/k8s_repository.go`、`README.md`
- 完成條件: conflict success、happy path no GET、non-conflict、exhaustion、reference change、opt-out、cancellation 測試通過；全套 race、vet、lint、格式與 diff 檢查通過。

### Protected Behavior

- workload authorization 仍在任何 external fetch 與 Kubernetes write 前完成。
- 每個 Secret 每輪只 fetch 一次；conflict 不重新 fetch。
- 成功遠端結果仍完整取代 `Secret.Data`，包含撤銷 missing keys 與空集合清除。
- fetch error 仍保留原 Data，未授權 reference 仍不 fetch/get/update。
- Worker parent cancellation 與 `syncInterval` deadline 繼續傳到所有 I/O。
- 正常第一次 update 成功時不增加 GET。
- runtime error/log 使用英文，不記錄 path、keys、Data 或機密值。

### 邊界

#### Allowed Changes

- `internal/syncer/application/sync_worker_usecase.go`
- `internal/syncer/application/sync_worker_usecase_test.go`
- `internal/syncer/infra/k8s_repository.go`，僅限必要的 error wrapping/contract 調整
- `README.md`
- `.specs/2026-08-04-14-14_BugFix-sync-update-conflict-retry/`

#### Forbidden

- 不修改 config schema、`configs/`、Vault/AWS clients、domain policy、Webhook、metrics schema、RBAC 或 deployment manifests。
- 不新增 Go module dependency或變更既有 module 版本。
- 不重試非 conflict error，不在 conflict 後重新 fetch external secret。
- 不用 `context.Background()` 脫離單輪 context，不用無 bounded wait 或 goroutine 實作 retry。
- 不以舊 Secret 物件重送，不 merge 遠端快照與最新本地 Data。
- 不將 `_workspace/` 納入提交。

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 以 red tests 固定 conflict 與安全契約 | 無 | Complete | 已證明目前不 retry |
| T2 擴充 repository interface 與測試替身 | T1 | Complete | concrete GetSecret 已存在 |
| T3 實作 context-aware conflict retry | T2 | Complete | 保留 happy path 無 GET |
| T4 實作 opt-in/reference revalidation | T3 | Complete | stale data fail closed |
| T5 更新操作文件 | T4 | Complete | 不宣稱持續 conflict 必然成功 |
| V1 驗收情境覆蓋 | T3、T4、T5 | Complete | 對應 requirements A-H |
| V2 回歸驗證 | V1 | Complete | 保護 snapshot、authorization、shutdown |
| V3 品質檢查清單 | V1、V2 | Complete | race、vet、lint、格式、Boundary |

## 實作任務

- [x] T1 以 red tests 固定 conflict 與安全契約
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/application/sync_worker_usecase_test.go`
    - Forbidden: 修改 production code；用純字串模擬 Kubernetes conflict；用 sleep 判斷 GET/update 是否開始
  - Depends: 無
  - Context: 擴充或新增 sequence-aware mock，使用 `apierrors.NewConflict` 建立真實 conflict。先加入 conflict 後成功、正常無 GET、非 conflict、exhaustion、reference changed、opt-out 與 cancellation 測試。測試需以 instance backoff 或 channel 控制，不能修改 package global。
  - Verify:
    - `go test ./internal/syncer/application -run 'TestSyncSecret_(RetriesConflictWithLatestSecret|SuccessDoesNotGetLatestSecret|NonConflictUpdateErrorDoesNotRetry|ConflictRetryExhausted|ReferenceChangedDuringConflictDoesNotUpdate|OptInRemovedDuringConflictDoesNotUpdate|ConflictRetryStopsOnContextCancellation)'`
    - T3 前，conflict success test 必須因只有一次 update 而失敗，證據記錄至 Implementation Notes。
    - `git diff --stat`、`git diff --check`

- [x] T2 擴充 repository interface 與測試替身
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/application/sync_worker_usecase.go` 的 `K8sSecretRepo`、`sync_worker_usecase_test.go` 的 mocks；必要時 `internal/syncer/infra/k8s_repository.go`
    - Forbidden: 新增第二套 Kubernetes client、改 constructor、下推 reference validation 到 infra、改 RBAC manifest
  - Depends: T1
  - Context: 在 application interface 新增 `GetSecret(ctx, namespace, name)`。concrete repository 已實作；所有 mocks 補齊明確行為。確認 `%w` wrapping 後 `apierrors.IsConflict` 與 NotFound 判斷仍成立。
  - Verify:
    - `go test ./internal/syncer/application ./internal/syncer/infra`
    - `go test ./cmd/vault-agent -run TestStartSyncWorker_CancelAndWait`
    - `go vet ./internal/syncer/application ./internal/syncer/infra`
    - `git diff --stat`、`git diff --check`

- [x] T3 實作 context-aware conflict retry
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/application/sync_worker_usecase.go`、必要的測試輔助調整
    - Forbidden: 使用非 context-aware sleep、重試 GET error/forbidden/not found、重新 fetch、增加 public API/config/global mutable backoff
  - Depends: T2
  - Context: constructor 初始化 private per-instance backoff 為 `retry.DefaultBackoff`。第一次 attempt 使用 list 物件；只有 conflict 後才 GET。以 `wait.ExponentialBackoffWithContext` 執行 attempts，保存 last conflict，exhaustion 時返回可辨識 cause。測試 instance 使用零延遲 backoff。
  - Verify:
    - `go test ./internal/syncer/application -run 'TestSyncSecret_(RetriesConflictWithLatestSecret|SuccessDoesNotGetLatestSecret|NonConflictUpdateErrorDoesNotRetry|ConflictRetryExhausted|ConflictRetryStopsOnContextCancellation)'`
    - `go test -race -count=1 ./internal/syncer/application`
    - `rg -n 'context\.Background|time\.Sleep' internal/syncer/application/sync_worker_usecase.go` 不得出現在 retry 路徑。
    - `git diff --stat`、`git diff --check`

- [x] T4 實作 opt-in/reference revalidation
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/application/sync_worker_usecase.go`、`sync_worker_usecase_test.go`
    - Forbidden: reference 改變後自動改抓新 path、繞過 authorization、在 error/log 輸出 path/keys/Data
  - Depends: T3
  - Context: conflict GET 後檢查最新 label 仍為 true，並 parse/比較 Backend、Path、Keys。不同、nil 或 parse error 都停止本輪。最新 metadata 保留，只以獨立 map 套用 desired Data；最新 Data 已相等時直接成功。
  - Verify:
    - `go test ./internal/syncer/application -run 'TestSyncSecret_(RetriesConflictWithLatestSecret|ReferenceChangedDuringConflictDoesNotUpdate|OptInRemovedDuringConflictDoesNotUpdate)'`
    - 人工檢查 runtime error/log 不含原始或最新 path、keys、Data。
    - `go test ./internal/syncer/application -run 'TestSyncOnce_(ReplacesDataAndRevokesMissingKeys|EmptyRemoteDataClearsSecret|FetchFailurePreservesData)'`
    - `git diff --stat`、`git diff --check`

- [x] T5 更新操作文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: `README.md`、本 spec Implementation Notes
    - Forbidden: 新增未實作 retry 設定、保證持續 conflict 一定成功、宣稱 conflict 會重新 fetch 外部機密
  - Depends: T4
  - Context: 在背景同步或可靠性說明補充 Kubernetes update conflict 會使用 bounded retry；reference/opt-in 改變時等待下一輪重新評估。
  - Verify:
    - `rg -n 'conflict|retry|重試|下一輪' README.md`
    - 人工比對 README 與實際 attempts、fetch 次數及 fail-closed 行為。

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Status: Complete
  - Boundary:
    - Allowed Changes: 任務邊界內測試、本 spec Implementation Notes
    - Forbidden: 以增加真實等待時間或移除 call count/resourceVersion/metadata 斷言掩蓋錯誤
  - Depends: T3、T4、T5
  - Context: 對照 requirements 場景 A-H，確認 happy path、conflict、error、security 與 cancellation 均有可執行測試。
  - Verify:
    - `go test ./internal/syncer/application -run 'TestSyncSecret_(RetriesConflictWithLatestSecret|SuccessDoesNotGetLatestSecret|NonConflictUpdateErrorDoesNotRetry|ConflictRetryExhausted|ReferenceChangedDuringConflictDoesNotUpdate|OptInRemovedDuringConflictDoesNotUpdate|ConflictRetryStopsOnContextCancellation)'`
    - `go test ./internal/syncer/application -run 'TestSyncOnce_(ReplacesDataAndRevokesMissingKeys|EmptyRemoteDataClearsSecret|FetchFailurePreservesData)|TestSyncWorker_CrossNamespaceReferenceDoesNotFetchOrUpdate'`
    - 測試明確斷言 fetch/get/update counts、latest resourceVersion、metadata preservation 與 error cause。

- [x] V2 回歸驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: 僅修正本 BugFix 直接造成的邊界內測試失敗，並先回填 Implementation Notes
    - Forbidden: 重寫 authorization、snapshot、Webhook、graceful shutdown、Vault/AWS client 或 metrics
  - Depends: V1
  - Context: 全套測試確認 repository interface 擴充與 retry 沒有破壞既有組裝及生命週期。
  - Verify:
    - `go test -race -count=1 ./...`
    - `go vet ./...`
    - 既有 `TestRun_CancelsInFlightFetchAndWaitsForExit`、`TestSyncOnce_ReplacesDataAndRevokesMissingKeys`、`TestSyncWorker_CrossNamespaceReferenceDoesNotFetchOrUpdate` 通過。

- [x] V3 品質檢查清單
  - Status: Complete
  - Boundary:
    - Allowed Changes: 格式修正、邊界內測試/文件、Implementation Notes
    - Forbidden: 為通過 lint 而移除 context、bounded attempts、reference validation 或 error cause 斷言
  - Depends: V1、V2
  - Context: 執行最終 race、vet、lint、格式、Boundary 與 diff 檢查，確認外部未追蹤產物未納入。
  - Verify:
    - `gofmt -l internal/syncer/application internal/syncer/infra` 無輸出。
    - `golangci-lint run ./...`
    - `git diff --stat`、`git diff --check`
    - `git status --short` 確認 `_workspace/` 仍未追蹤。
  - 品質項目:
    - [x] happy path update 不增加 GET
    - [x] 只有 conflict 進入 retry
    - [x] conflict 後使用最新 resourceVersion
    - [x] 最新 labels/annotations/metadata 被保留
    - [x] desired Data 仍是完整快照，不 merge 本地 key
    - [x] reference 改變時不寫入 stale data
    - [x] opt-in 移除時不再 update
    - [x] conflict 不重新 fetch external secret
    - [x] exhaustion 保留 conflict cause
    - [x] backoff、GET、update 遵守 context
    - [x] 無 package global mutable backoff
    - [x] authorization 與 graceful shutdown 回歸通過
    - [x] runtime error/log 為英文且不含機密值
    - [x] README 與實作一致
    - [x] `git diff --check` 通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的 `Execution Context`
2. 目前未完成 task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個 `.specs` 目錄。若文件很大，先用標題與關鍵字定位：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-14-14_BugFix-sync-update-conflict-retry
```

## Implementation Notes

- 2026-08-04: 已建立規劃，尚未修改 production code 或測試。
- 2026-08-04: T1 red tests 確認現行實作遇到 conflict 只 update 一次且 GET 為零；conflict success、exhaustion、reference changed、opt-out 與 cancellation 測試均按預期失敗。
- 2026-08-04: T2 application repository interface 已加入 `GetSecret`；concrete repository 無需修改，既有 application/main 測試與 vet 通過。
- 2026-08-04: T3 使用 per-instance `retry.DefaultBackoff` 與 `ExponentialBackoffWithContext` 完成 bounded retry；wrapped conflict、happy path、非 conflict、exhaustion 與 cancellation 測試通過。
- 2026-08-04: T4 conflict 後會重驗 opt-in 與 Backend/Path/Keys，並以最新物件為基底只替換 Data；安全情境與 application race test 通過。
- 2026-08-04: T5 README 已說明 bounded conflict retry、單次 external fetch 與契約改變時延後至下一輪。
- 2026-08-04: 初次 lint 發現 `wait.ErrWaitTimeout` deprecated；依 library 指引改用 `wait.Interrupted`，並明確保留 cancellation/deadline cause。
- 2026-08-04: V1 requirements 場景 A-H 全數通過，並斷言 fetch/get/update counts、最新 resourceVersion、metadata 與 error cause。
- 2026-08-04: V2 `go test -race -count=1 ./...` 與 `go vet ./...` 通過，快照、authorization、Webhook 與 graceful shutdown 無回歸。
- 2026-08-04: V3 `golangci-lint run ./...` 回報 0 issues；格式、Boundary 與 diff 檢查通過。
- concrete `K8sRepository.GetSecret` 已存在；application interface 與 mocks 尚未包含。
- client-go `RetryOnConflict` 使用非 context-aware wait；本設計改採相同 `DefaultBackoff` 參數搭配 `ExponentialBackoffWithContext`。
- conflict 後必須重驗 opt-in/reference，避免將舊 path 的 snapshot 寫入已改變契約的 Secret。

## 驗證結果摘要

- 新行為驗證: conflict retry、安全重驗與 cancellation 目標測試全部通過
- 回歸驗證: 全套 race test、vet 與 lint 通過
- 文件一致性: README、requirements、design、tasks 與 4 attempts、單次 fetch、fail-closed 行為一致
- 剩餘風險: 持續 conflict 仍可能耗盡；reference 改變時一致性延後到下一輪

## 後續改善

- [ ] 另案評估以 patch 或 server-side apply 明確管理 `Secret.Data` ownership。
- [ ] 另案評估 conflict retry metrics 與告警，但避免高基數 label。
- [ ] 另案評估 `terminationGracePeriodSeconds` 與 30 秒 shutdown budget 對齊。
