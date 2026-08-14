# 任務文件：Sync Worker Graceful Shutdown

Status: Complete

## Execution Context

- 意圖: 讓 Sync Worker 的 in-flight list/fetch/update 繼承 shutdown cancellation，取消後停止剩餘 Secret，並讓 main 在既有 30 秒 graceful cleanup budget 內等待 Worker 結束。
- 非目標: 不新增 timeout 設定；不改 graceful library、infra clients、Secret 快照、authorization、retry、RBAC 或 deployment。
- 已定決策: 每輪使用 `context.WithTimeout(parent, syncInterval)`；main 持有 Worker cancel/done；task 所有 return path 都 cancel；cleanup 依 LIFO 實現 HTTP、Worker wait、tracer 順序；共用現有 30 秒 budget。
- 邊界: 變更限於 Sync Worker application 實作/測試、main lifecycle 實作/測試、`README.md` 與本 spec。
- 關鍵檔案: `internal/syncer/application/sync_worker_usecase.go`、`internal/syncer/application/sync_worker_usecase_test.go`、`cmd/vault-agent/main.go`、`cmd/vault-agent/main_test.go`、`README.md`
- 完成條件: in-flight list/fetch 取消、remaining secrets stop、interval deadline 保留、main cancel/wait 與 wait timeout 測試通過；全套 race、vet、lint、格式與 diff 檢查通過。

### Protected Behavior

- Worker 仍等待第一次 ticker，不在啟動時立即同步。
- 每輪仍受 `syncInterval` deadline 限制。
- Secret 完整快照取代、空集合與撤銷語意不變。
- authorization deny 仍在 fetch 前發生。
- HTTP `Shutdown(ctx)`、fallback `Close()` 與 tracer cleanup 保留。
- runtime error 與 log 訊息使用英文，不記錄機密值。

### 邊界

#### Allowed Changes

- `internal/syncer/application/sync_worker_usecase.go`
- `internal/syncer/application/sync_worker_usecase_test.go`
- `cmd/vault-agent/main.go`
- `cmd/vault-agent/main_test.go`
- `README.md`
- `.specs/2026-08-04-13-50_BugFix-sync-worker-graceful-shutdown/`

#### Forbidden

- 不修改 `github.com/vincent119/commons/graceful` 或 Go module 版本。
- 不修改 config schema、`configs/`、infra clients、domain policy、Webhook、metrics schema、RBAC 或 deployment manifests。
- 不新增 goroutine 無 bounded wait，不用 sleep 作為 lifecycle 同步機制。
- 不為通過 shutdown 測試而移除正常 `syncInterval` deadline。
- 不改變 Secret Data、authorization、error 英文化或 HTTP 既有契約。
- 不將 `_workspace/` 或 `skills-lock.json` 納入提交。

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 以 blocking mocks 固定 Worker cancellation 契約 | 無 | Complete | 已確認 detached context bug |
| T2 傳遞 parent cancellation 並停止本輪 | T1 | Complete | 保留 interval deadline |
| T3 在 main 管理 Worker cancel、done 與 bounded wait | T2 | Complete | 不修改 graceful library |
| T4 更新優雅關機文件 | T3 | Complete | 不誤稱不合作 backend 必然完成 |
| V1 驗收情境覆蓋 | T2、T3、T4 | Complete | 對應 requirements 全部情境 |
| V2 回歸驗證 | V1 | Complete | 保護 snapshot、authorization、HTTP 與 tracer |
| V3 品質檢查清單 | V1、V2 | Complete | 完成 race、lint、格式、Boundary 與 diff 檢查 |

## 實作任務

- [x] T1 以 blocking mocks 固定 Worker cancellation 契約
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/application/sync_worker_usecase_test.go`
    - Forbidden: 修改 production code；使用無上限 channel receive；用 sleep 判斷 I/O 是否已開始
  - Depends: 無
  - Context: 新增 blocking repository/fetcher，以 `started` channel 確認 I/O 已開始，以 context cancellation 作為返回條件。測試必須有失敗 timeout 與清理路徑，避免現有 detached context 留下 goroutine。
  - Verify:
    - `go test ./internal/syncer/application -run 'TestRun_CancelsInFlight(Fetch|List)AndWaitsForExit|TestSyncOnce_CancellationStopsRemainingSecrets'`
    - T2 前，in-flight list/fetch 測試必須因收不到 parent cancellation 而失敗，並將證據記錄至 Implementation Notes。
    - `go test -race ./internal/syncer/application -run 'TestRun_CancelsInFlight'`

- [x] T2 傳遞 parent cancellation 並停止本輪
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/application/sync_worker_usecase.go`、必要的 `sync_worker_usecase_test.go` 測試輔助調整
    - Forbidden: 改動 public 介面、SecretFetcher/K8s repo 契約、snapshot/authorization 邏輯、metrics schema；移除 interval deadline
  - Depends: T1
  - Context: 以 `context.WithTimeout(ctx, uc.syncInterval)` 取代 detached context。`syncOnce` 在 list error、每個 Secret 前與 `syncSecret` error 後檢查 `ctx.Err()`，shutdown cancellation 直接返回。只有 parent cancellation 不增加 sync error metric，interval deadline 仍保留可觀測錯誤。
  - Verify:
    - `go test ./internal/syncer/application -run 'TestRun_CancelsInFlight(Fetch|List)AndWaitsForExit|TestSyncOnce_CancellationStopsRemainingSecrets|TestRun_SyncRoundStillHasIntervalDeadline'`
    - `go test -race -count=1 ./internal/syncer/application`
    - `rg -n 'WithTimeout\(context\.Background' internal/syncer/application/sync_worker_usecase.go` 無輸出。

- [x] T3 在 main 管理 Worker cancel、done 與 bounded wait
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/main.go`、新增 `cmd/vault-agent/main_test.go`
    - Forbidden: 改動 HTTP route/server timeout、graceful timeout、tracer 實作、backend 組裝、新增 public API 或外部 dependency
  - Depends: T2
  - Context: 以最小 package-private `syncWorkerRunner` 介面建立 start/cancel/done helper。task 以 defer 保證 signal 與 server error return 都 cancel。註冊 Worker wait cleaner，使用同一 cleanup context；依 LIFO 讓 HTTP 先關閉、Worker 再 wait、tracer 最後 shutdown。
  - Verify:
    - `go test ./cmd/vault-agent -run 'Test(StartSyncWorker_CancelAndWait|WaitForSyncWorker_DeadlineExceeded)'`
    - `go test -race -count=1 ./cmd/vault-agent`
    - 程式碼檢查 task 的 server error 與 `ctx.Done()` 路徑共用 Worker cancel，cleaner 沒有無 bounded receive。

- [x] T4 更新優雅關機文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: `README.md`、本 spec Implementation Notes
    - Forbidden: 聲稱不合作 backend 必然完成、新增未實作設定或更改 deployment 指引
  - Depends: T3
  - Context: 將「安全排空請求後退出」擴充為 HTTP 停止、Worker context 取消/等待與 tracer cleanup，明確三者共用 30 秒 budget。
  - Verify:
    - `rg -n '30 秒|Worker|context|等待|tracer' README.md`
    - 人工比對 README 與 `main.go` cleaner 註冊順序。

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Status: Complete
  - Boundary:
    - Allowed Changes: 任務邊界內測試、本 spec Implementation Notes
    - Forbidden: 以延長 timeout 或移除 channel 斷言掩蓋 lifecycle bug
  - Depends: T2、T3、T4
  - Context: 確認 requirements 的 in-flight list/fetch、remaining secrets stop、main cancel/wait、wait deadline 與既有行為都有可執行驗證。
  - Verify:
    - `go test ./internal/syncer/application -run 'TestRun_CancelsInFlight(Fetch|List)AndWaitsForExit|TestSyncOnce_CancellationStopsRemainingSecrets|TestRun_SyncRoundStillHasIntervalDeadline'`
    - `go test ./cmd/vault-agent -run 'Test(StartSyncWorker_CancelAndWait|WaitForSyncWorker_DeadlineExceeded)'`
    - 測試使用 started/done channel，只將 timeout 作為失敗上限。

- [x] V2 回歸驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: 僅修正由本 BugFix 直接造成的邊界內測試失敗，並先回填 Implementation Notes
    - Forbidden: 重寫 snapshot、authorization、HTTP、tracer、infra clients 或 graceful library
  - Depends: V1
  - Context: 全套測試確認 cancellation 修正沒有破壞 Secret 同步與 Webhook/process lifecycle。
  - Verify:
    - `go test -race -count=1 ./...`
    - `go vet ./...`
    - 既有 `TestRun_StopsOnContextCancellation`、`TestSyncOnce_ReplacesDataAndRevokesMissingKeys`、`TestSyncWorker_CrossNamespaceReferenceDoesNotFetchOrUpdate` 通過。

- [x] V3 品質檢查清單
  - Status: Complete
  - Boundary:
    - Allowed Changes: 格式修正、邊界內測試/文件、Implementation Notes
    - Forbidden: 為通過 lint 而移除 context 傳遞、bounded wait 或 existing behavior 斷言
  - Depends: V1、V2
  - Context: 執行最終 race、vet、lint、格式、Boundary 與 diff 檢查，確認未追蹤外部產物未納入。
  - Verify:
    - `gofmt -l cmd/vault-agent internal/syncer/application` 無輸出。
    - `golangci-lint run ./...`
    - `git diff --stat`、`git diff --check`
    - `git status --short` 確認 `_workspace/` 與 `skills-lock.json` 仍未追蹤。
  - 品質項目:
    - [x] in-flight list 收到 parent cancellation
    - [x] in-flight fetch 收到 parent cancellation
    - [x] 取消後不處理剩餘 Secret
    - [x] 取消後不 Update Secret
    - [x] 正常 interval deadline 仍存在
    - [x] shutdown cancellation 不增加一般 sync error metric
    - [x] main task 所有 return path 取消 Worker
    - [x] main 在 process exit 前等待 Worker done
    - [x] Worker wait 遵守 cleanup deadline
    - [x] cleanup 順序為 HTTP、Worker、tracer
    - [x] Secret snapshot 與 authorization 回歸通過
    - [x] runtime error/log 維持英文且無機密值
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
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-13-50_BugFix-sync-worker-graceful-shutdown
```

## Implementation Notes

- 2026-08-04: 已建立規劃，尚未執行 production code 與測試變更。
- 2026-08-04: T1 red test 已確認 detached context 問題：in-flight list/fetch 收到 `context deadline exceeded` 而非 parent cancellation，且取消第一個 Secret 後仍呼叫第二次 fetch。
- 2026-08-04: T2 改以 parent context 建立單輪 timeout；目標測試及 application package race test 通過，並確認 interval deadline 仍存在。
- 2026-08-04: T3 新增 package-private Worker lifecycle helper；task 以 defer 在 signal 與 server error 路徑取消 Worker，cleanup wait 受共用 30 秒 context 限制。main package 目標測試與 race test 通過。
- 2026-08-04: T4 更新 README，記錄 HTTP、Worker、tracer 的關閉順序與共用 30 秒 budget，未宣稱忽略 context 的 backend 必然及時完成。
- 2026-08-04: V1 所有 application/main 目標驗收測試通過；測試以 started/done channel 同步，timeout 僅作失敗上限。
- 2026-08-04: V2 `go test -race -count=1 ./...` 與 `go vet ./...` 通過，Secret 快照、authorization、Webhook 與既有 Worker cancellation 無回歸。
- 2026-08-04: V3 `golangci-lint run ./...` 回報 0 issues；格式、Boundary 與 diff 檢查通過。
- 修正前 detached context 位於 `SyncWorkerUseCase.Run` 的 `context.WithTimeout(context.Background(), uc.syncInterval)`。
- 修正前 main task 以 `go syncWorker.Run(ctx)` 啟動 Worker，沒有 done channel 或 wait cleaner。
- `commons/graceful` v0.2.4 在 task 返回後建立 30 秒 cleanup context，並以 LIFO 執行 cleaners；本次設計依賴此已驗證契約。

## 驗證結果摘要

- 新行為驗證: application 與 main 的指定驗收測試全部通過
- 回歸驗證: 全套 race test、vet 與 lint 通過
- 文件一致性: README 已反映 task 先取消 Worker，以及 cleanup 依 HTTP、Worker wait、tracer 的順序共用 30 秒 budget
- 剩餘風險: 第三方 backend 若忽略 context，Worker 只能由 cleanup deadline 限制等待

## 後續改善

- [ ] 另案評估 Kubernetes Update conflict retry 與 backoff。
- [ ] 另案評估將 Worker 改為可回傳 error 的 errgroup task。
- [ ] 另案評估 deployment `terminationGracePeriodSeconds` 與全域 cleanup budget 的對齊。
