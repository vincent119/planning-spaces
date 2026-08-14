# 需求文件：Sync Worker Graceful Shutdown

## 來源

- Draft: 無
- Type: BugFix
- Owner: Vincent
- Status: Complete

## 文件定位

本 spec 接續 `.specs/2026-08-04-10-52_Feature-secure-secret-access/` 與 `.specs/2026-08-04-11-43_BugFix-secret-key-revocation-semantics/` 的 Sync Worker 實作，處理兩份文件明確留待後續的 Worker shutdown context 契約。本次只修正進行中同步的取消傳遞與主程式等待邏輯，不重寫 Secret 快照對帳、authorization、Vault/AWS client、HTTP server 或外部 graceful library。

參考來源：

- 需求來源: 使用者確認下一項處理 `BugFix-sync-worker-graceful-shutdown`
- 既有文件: `README.md`、`.specs/2026-08-04-11-43_BugFix-secret-key-revocation-semantics/tasks.md`
- 既有程式碼: `internal/syncer/application/sync_worker_usecase.go`、`cmd/vault-agent/main.go`
- 外部契約: `github.com/vincent119/commons/graceful` v0.2.4 的 `Run`、`WithCleanup` 與 `WithTimeout`

## 背景

`SyncWorkerUseCase.Run` 會監聽傳入 context，但每輪同步卻以 `context.Background()` 建立獨立 timeout context。當程式收到 `SIGINT` 或 `SIGTERM`，父 context 會取消，但進行中的 Kubernetes list、Vault/AWS fetch 或 Secret update 不會收到取消，最長可持續到 `sync.interval_seconds`。

主程式又以 goroutine 啟動 Worker，task 收到 signal 後立即返回並進入 HTTP/tracer cleanup，沒有等待 Worker 結束。因此現有「30 秒優雅關機」只覆蓋 cleanup，沒有對 Worker 建立可驗證的取消與等待契約。

## 問題陳述

關機期間進行中的 Sync Worker I/O 與父 context 脫離，主程式也不等待 Worker 結束。這可能讓 process 在同步尚未完成時退出，或讓取消後繼續發出外部機密與 Kubernetes API 呼叫。

## 目標

1. 父 context 取消必須立即傳遞到進行中的 list、fetch 與 update。
2. 取消後不得再開始下一個 Secret 或下一輪同步。
3. 主程式在 signal 或 server fatal error 後必須取消 Worker，並在 process 退出前等待 Worker 結束。
4. Worker 等待必須受既有 30 秒 graceful cleanup context 限制，backend 不遵守 context 時不可無限阻塞。
5. 關機造成的 `context.Canceled` 不視為普通 sync failure，不繼續處理剩餘 Secret。
6. 保留正常 interval timeout、Secret 快照取代、authorization 與 HTTP graceful shutdown 行為。

## 非目標

1. 不新增 shutdown timeout 或 per-operation timeout 設定鍵。
2. 不修改 `github.com/vincent119/commons/graceful` 套件。
3. 不實作 Kubernetes Update conflict retry、backend retry、queue drain 或 checkpoint/resume。
4. 不改變 Worker 啟動時點；仍等待第一次 ticker，不在啟動時立即 sync。
5. 不修改 SecretFetcher、K8s repository、policy、RBAC、Webhook API 或 deployment manifest。
6. 不保證不遵守 context 的第三方 SDK 能完成工作；只保證等待有上限。

## 已定決策

- 每輪 context 使用 `context.WithTimeout(parent, syncInterval)`，同時繼承父取消與保留原 interval deadline。
- `syncOnce` 在 list 後與每個 Secret 之間檢查 context，取消後直接停止本輪。
- 主程式為 Worker 建立可明確 cancel 的 child context 與 done channel。
- task 因 signal 或 server error 返回時都取消 Worker。
- graceful cleanup 先停止 HTTP servers，再等待 Worker done，最後關閉 tracer；三者共用既有 30 秒 budget。
- Worker 等待超時會回傳英文 error 並由 graceful manager 彙總，但不阻止後續 cleanup 被呼叫。
- 關機 context 取消時不增加 sync error metric；正常 interval deadline 與其他 backend error 仍保留錯誤計數。

## 待確認項目

- 無。本次沿用既有 30 秒 shutdown budget，不新增使用者設定。

## 現有行為

1. `Run` 只在閒置的外層 select 看到 `ctx.Done()` 時退出。
2. 進行中的 `syncOnce` 使用從 `context.Background()` 建立的 context，不受關機 signal 影響。
3. 單個 fetch 取消後，loop 仍可繼續處理其他 Secret。
4. main task 啟動 Worker goroutine 後不保留 done channel，cleanup 只處理 HTTP server 與 tracer。
5. 既有 `TestRun_StopsOnContextCancellation` 只測試第一次 ticker 前的閒置取消，沒有覆蓋 in-flight I/O。

## 新行為

1. 父 context 取消時，本輪 context 同步取消，所有正在使用該 context 的 repository/fetcher 可立即返回。
2. `syncOnce` 看到 context 取消後不記錄一般 per-secret failure，不更新 Secret，不處理後續 Secret。
3. `Run` 只在進行中同步已因 context 結束後返回。
4. main task 不論因 signal 或 server error 結束，都會取消 Worker child context。
5. cleanup 在關閉 HTTP servers 後等待 Worker done；Worker 已結束時立即通過，未結束時最長等待 cleanup context deadline。
6. tracer 在 Worker 結束後關閉，避免遺失 Worker 最後的 trace/log lifecycle。

## 影響範圍

- 使用者: 部署 Vault-Agent 的平台管理者
- 功能: Sync Worker 取消、process shutdown、HTTP/tracer/Worker cleanup 順序
- API / CLI: 無；不新增設定鍵或對外 endpoint
- Data / Storage: 無 schema 變更；取消後不建立新 Secret update
- 文件 / 安裝 / 發布: 更新 `README.md` 的優雅關機說明；無 manifest 變更

## 使用情境

- 作為平台管理者，我想要 Vault-Agent 收到 SIGTERM 後取消進行中同步並等待 Worker 結束，以便 Kubernetes 能在 termination grace period 內可預測地關閉 Pod。
- 作為安全工程師，我想要 shutdown 後不再啟動其他機密讀取，以便關機邊界後沒有額外外部存取。

## 驗收情境

### 情境：取消進行中的 fetch

- 場景：Worker 已進入一輪同步並阻塞在 SecretFetcher
- 測試：`TestRun_CancelsInFlightFetchAndWaitsForExit`
- 假設：fetcher 等待 `ctx.Done()` 才返回，且已用 channel 回報 fetch 開始
- 當：取消傳入 `Run` 的父 context
- 那麼：fetcher 收到 `context.Canceled`，Worker 在限時內返回，Update 次數為零

### 情境：取消進行中的 Kubernetes list

- 場景：Worker 阻塞在 `ListSecretsByLabel`
- 測試：`TestRun_CancelsInFlightListAndWaitsForExit`
- 假設：repository 等待 `ctx.Done()` 才返回
- 當：取消父 context
- 那麼：repository 收到取消，Worker 在限時內返回，不進入 fetch/update

### 情境：取消後不處理剩餘 Secret

- 場景：list 回傳多個 Secret，第一個 fetch 進行中收到取消
- 測試：`TestSyncOnce_CancellationStopsRemainingSecrets`
- 假設：fetcher 在第一次呼叫等待 context 取消
- 當：取消 `syncOnce` context
- 那麼：fetch 總次數為一，不呼叫 Update，第二個 Secret 不被處理

### 情境：main 取消並等待 Worker

- 場景：main task 收到 shutdown 或 server error
- 測試：`TestStartSyncWorker_CancelAndWait`
- 假設：測試 Worker 在 context 取消後關閉 done
- 當：呼叫 Worker cancel，再使用 cleanup context 等待
- 那麼：Worker 確實收到取消，等待成功且無 goroutine 洩漏

### 情境：不合作 Worker 不會無限阻塞 cleanup

- 場景：Worker done 在 cleanup deadline 前不會關閉
- 測試：`TestWaitForSyncWorker_DeadlineExceeded`
- 假設：傳入很短 deadline 的 cleanup context
- 當：等待 Worker done
- 那麼：在 deadline 附近返回可用 `errors.Is` 判斷的 `context.DeadlineExceeded`

### 情境：既有行為不被破壞

- 場景：正常 ticker 同步、快照取代、授權拒絕、HTTP shutdown 與 tracer cleanup
- 測試：`TestRun_StopsOnContextCancellation|TestSyncOnce_ReplacesDataAndRevokesMissingKeys|TestSyncWorker_CrossNamespaceReferenceDoesNotFetchOrUpdate`
- 假設：使用既有測試與 30 秒 graceful timeout
- 當：執行全套測試
- 那麼：原有快照、authorization、HTTP 與 telemetry 行為保持不變

## 驗收條件

1. in-flight list 與 fetch 都能在父 context 取消後收到 `context.Canceled`。
2. 取消後不處理下一個 Secret，不呼叫 Update。
3. main 層有可測試的 Worker cancel/done/wait 契約，並在 server error 與 signal 路徑共用。
4. Worker 等待遵守 cleanup context deadline，不合作 Worker 不會無限阻塞。
5. cleanup 順序為 HTTP servers、Worker wait、tracer，且不新增第二個 timeout 設定。
6. `go test -race -count=1 ./...`、`go vet ./...`、`golangci-lint run ./...` 與 `git diff --check` 全部通過。

## 驗證需求

- Unit / Integration: `go test ./internal/syncer/application -run 'TestRun_CancelsInFlight|TestSyncOnce_CancellationStopsRemainingSecrets'`、`go test ./cmd/vault-agent -run 'Test(StartSyncWorker_CancelAndWait|WaitForSyncWorker_DeadlineExceeded)'`
- CLI / Dry-run: 無
- 文件檢查: 確認 `README.md` 的優雅關機說明包含 Worker 取消與等待
- 回歸驗證: `go test -race -count=1 ./...`、`go vet ./...`、`golangci-lint run ./...`

## 風險與假設

| 類型 | 內容 | 處理方式 |
|------|------|----------|
| 風險 | Worker 等待耗盡全部 cleanup budget | 先取消 Worker，HTTP cleanup 先執行，Worker wait 使用共用 deadline 並回傳 timeout error |
| 風險 | 取消被記為 sync failure 造成告警雜訊 | 以 `ctx.Err()` 辨識 shutdown cancellation，不增加錯誤 metric、不繼續 per-secret loop |
| 風險 | main cleanup closure 在 Worker 未啟動時等待 nil channel | 無 Worker 時不註冊 wait cleaner，helper 對 nil done fail fast |
| 風險 | 測試使用 sleep 造成 flaky | 用 started/done channel 建立 happens-before，只用短 timeout 作為失敗上限 |
| 假設 | Kubernetes、Vault 與 AWS client 會遵守傳入 context | 以 blocking mock 驗證 application 傳遞；不合作實作由 cleanup deadline 限制等待 |

## 摘要

- 關鍵決策: 每輪 context 繼承父取消；main 持有 cancel/done；HTTP 停止後在既有 30 秒 budget 內等待 Worker；tracer 最後關閉
- 待確認項目: 無
- 風險: 不合作 backend 可用完 cleanup budget，但不會無限阻塞
- 下一步: 審閱 `design.md` 與 `tasks.md`，確認後依任務順序實作
