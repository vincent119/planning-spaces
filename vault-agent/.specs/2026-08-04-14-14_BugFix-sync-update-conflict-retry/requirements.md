# 需求文件：Sync Update Conflict Retry

## 來源

- Draft: 無
- Type: BugFix
- Owner: Vincent
- Status: Complete

## 文件定位

本 spec 接續 `.specs/2026-08-04-11-43_BugFix-secret-key-revocation-semantics/` 與 `.specs/2026-08-04-13-50_BugFix-sync-worker-graceful-shutdown/`，修正 Sync Worker 更新 Kubernetes Secret 時遇到 optimistic concurrency conflict 只能等待下一輪的問題。本次保留已完成的完整快照刪鍵語意、workload authorization 與 graceful shutdown，不重寫 Vault/AWS client、Webhook、policy 或外部 graceful manager。

參考來源：

- 需求來源: 使用者確認下一階段處理 Kubernetes Secret Update conflict retry
- 既有程式碼: `internal/syncer/application/sync_worker_usecase.go`、`internal/syncer/infra/k8s_repository.go`
- Kubernetes 契約: `k8s.io/client-go/util/retry.DefaultBackoff` 與 `k8s.io/apimachinery/pkg/util/wait.ExponentialBackoffWithContext`
- 既有 repository 能力: concrete `K8sRepository` 已有 `GetSecret`，application `K8sSecretRepo` 尚未暴露該方法

## 背景

Sync Worker 從 list 結果取得 Secret，抓取遠端機密並呼叫 `UpdateSecret`。若其他 controller 或使用者在 list 與 update 之間修改同一個 Secret，Kubernetes API 會因 `resourceVersion` 過期回傳 `409 Conflict`。目前 Worker 將該次更新視為一般失敗，不重讀最新物件，只能等待下一個 `sync.interval_seconds`。

直接重送舊 Secret 不安全：它可能覆蓋其他 actor 的 labels、annotations 或 metadata。若 conflict 期間 backend/path/keys 已變更，把先前取得的遠端資料套用到最新物件，還可能造成跨機密引用的 stale data write。因此 retry 必須重新讀取最新物件、保留最新 metadata，並在寫入前確認同步 opt-in 與 parsed reference 未改變。

## 現有行為

- 每輪 list 後，以列出的 Secret 物件進行一次 `UpdateSecret`。
- `UpdateSecret` 包裝 Kubernetes API error，但 application 不辨識 conflict。
- conflict、forbidden、not found、網路錯誤都不重試。
- 更新失敗後由 `syncOnce` 記錄錯誤，該 Secret 等待下一輪。
- concrete repository 已支援 `GetSecret(ctx, namespace, name)`，但 `K8sSecretRepo` 介面只有 list/update。

## 新行為

- 正常第一次 update 成功時不增加 GET。
- 第一次 update 回傳 Kubernetes `409 Conflict` 時，進入 bounded retry。
- 每次 conflict 後重新 GET 最新 Secret，保留最新 `resourceVersion`、labels、annotations 與其他 metadata，只取代 `Data` 為已授權且成功抓取的完整遠端快照。
- retry 前確認最新 Secret 仍有同步 opt-in label，且 parsed backend/path/keys 與本輪原始 reference 相同。
- opt-in 移除、reference 缺失、無法解析或已變更時，不寫入 stale data，結束本次同步並交由下一輪重新評估、授權與 fetch。
- 非 conflict error 立即返回，不進行 GET 或 retry。
- retry 次數耗盡時返回最後一個 conflict，讓既有 error handling 記錄失敗。
- retry、GET 與 update 全程遵守單輪 context；shutdown cancellation 或 interval deadline 發生後停止等待與後續 I/O。

## 目標

- 降低短暫 Kubernetes update conflict 導致 Secret 長時間不一致的機率。
- 避免重送 stale object 覆蓋其他 actor 的 metadata。
- 避免 secret reference 改變後寫入舊路徑的機密資料。
- 保留 context cancellation、單輪 deadline 與既有完整快照語意。
- 以固定 bounded backoff 實作，不新增設定或 dependency。

## 非目標

- 不處理非 conflict error 的通用 retry。
- 不重試 Vault/AWS fetch、Kubernetes list 或 admission mutation。
- 不新增 retry 次數、間隔或 jitter 的 config/env。
- 不新增 metrics schema、Prometheus label 或告警規則。
- 不改用 server-side apply、JSON Patch 或 UpdateStatus。
- 不重新 fetch 遠端機密；reference 改變時等待下一輪重新授權與 fetch。
- 不修改 RBAC、deployment、`terminationGracePeriodSeconds` 或 graceful timeout。
- 不解決多個合法 writer 對同一個 `Secret.Data` 持續競爭的 ownership 問題。

## 使用情境

### 情境一：短暫 conflict 後成功

其他 controller 在 Worker 首次 update 前修改 metadata，Worker 收到 conflict。Worker 重新 GET 最新 Secret，確認同步契約未變，只替換 Data，再以最新 `resourceVersion` 更新成功。

### 情境二：非 conflict error

Kubernetes API 回傳 forbidden、not found 或網路錯誤。Worker 不執行 GET/retry，直接由既有流程記錄失敗。

### 情境三：reference 在 conflict 期間變更

使用者將 path、backend 或 keys 改成另一個引用。Worker 不得把舊 reference 抓到的資料寫入最新 Secret，必須終止本次更新，讓下一輪重新授權與 fetch。

### 情境四：同步 opt-in 被移除

最新 Secret 已移除 `inject-vault-agent=true` label。Worker 停止管理該物件，不執行第二次 update。

### 情境五：關機或單輪逾時

Worker 在 conflict retry 的 GET、update 或 backoff 期間收到 context cancellation/deadline。retry 立即停止，不開始後續 I/O。

## 驗收情境

### 場景 A：conflict 後保留最新 metadata 並成功更新

- 測試: `TestSyncSecret_RetriesConflictWithLatestSecret`
- 假設: 首次 update 回傳 Kubernetes conflict，GET 回傳相同 reference、較新 resourceVersion 與其他 actor 新增的 label
- 當: Worker 執行 conflict retry
- 那麼: 遠端 fetch 只發生一次、GET 發生一次、第二次 update 使用最新 resourceVersion 與 label，Data 等於遠端完整快照

### 場景 B：正常更新不增加 GET

- 測試: `TestSyncSecret_SuccessDoesNotGetLatestSecret`
- 假設: 第一次 update 成功
- 當: Worker 同步不同的 Data
- 那麼: update 一次且 GET 為零次

### 場景 C：非 conflict error 不重試

- 測試: `TestSyncSecret_NonConflictUpdateErrorDoesNotRetry`
- 假設: 第一次 update 回傳 forbidden
- 當: Worker 同步 Secret
- 那麼: 返回原錯誤、update 一次且 GET 為零次

### 場景 D：retry 耗盡

- 測試: `TestSyncSecret_ConflictRetryExhausted`
- 假設: 每次 update 都回傳 conflict，GET 持續回傳相同 reference 的最新物件
- 當: 達到 bounded backoff 的最大 attempts
- 那麼: 停止重試並保留可由 `apierrors.IsConflict` 辨識的最後錯誤

### 場景 E：reference 改變時 fail closed

- 測試: `TestSyncSecret_ReferenceChangedDuringConflictDoesNotUpdate`
- 假設: 首次 update conflict，GET 的最新 Secret 改成另一個 backend/path/keys
- 當: Worker 準備 retry
- 那麼: 不執行第二次 update、不重新 fetch，並返回明確英文錯誤

### 場景 F：opt-in 移除時停止管理

- 測試: `TestSyncSecret_OptInRemovedDuringConflictDoesNotUpdate`
- 假設: 首次 update conflict，GET 的最新 Secret 不再有同步 label
- 當: Worker 準備 retry
- 那麼: 不執行第二次 update，並返回明確英文錯誤

### 場景 G：retry 遵守 cancellation

- 測試: `TestSyncSecret_ConflictRetryStopsOnContextCancellation`
- 假設: 首次 update conflict，後續 GET 已開始且阻塞等待 context
- 當: parent context 被取消
- 那麼: GET 收到 `context.Canceled`、沒有第二次 update、同步立即返回 cancellation

### 場景 H：既有快照與 authorization 不變

- 測試: `TestSyncOnce_ReplacesDataAndRevokesMissingKeys`、`TestSyncWorker_CrossNamespaceReferenceDoesNotFetchOrUpdate`
- 假設: 遠端成功回傳完整快照，或 workload reference 未獲授權
- 當: 執行同步
- 那麼: 缺失 key 仍被刪除；未授權 reference 仍不 fetch、不 GET、不 update

## 驗收條件

- [x] `K8sSecretRepo` 能取得指定 namespace/name 的最新 Secret，concrete repository 沿用既有 `GetSecret`。
- [x] 只有 Kubernetes conflict 進入 retry；其他 error 不 retry。
- [x] 第一次 update 成功時沒有額外 GET。
- [x] conflict 後每次 update 都使用重新 GET 的最新 resourceVersion 與 metadata。
- [x] retry 只替換 `Secret.Data`，不回寫 stale labels/annotations。
- [x] 最新 opt-in 或 parsed reference 改變時不寫入 stale data。
- [x] retry 耗盡後錯誤仍可用 `apierrors.IsConflict` 辨識。
- [x] retry backoff bounded 且可被 context cancellation/deadline 中斷。
- [x] remote fetch 每輪每個 Secret 最多一次；conflict 不觸發重新 fetch。
- [x] runtime error/log 使用英文，不記錄 Secret data 或機密值。
- [x] Secret 完整快照、刪鍵、空集合、authorization 與 graceful shutdown 回歸通過。

## 影響範圍

- `internal/syncer/application/sync_worker_usecase.go`
- `internal/syncer/application/sync_worker_usecase_test.go`
- `internal/syncer/infra/k8s_repository.go` 僅在既有 `GetSecret` 契約需要調整錯誤保留時修改
- `README.md` 若需說明 conflict retry 行為
- 本 spec 文件

## 驗證需求

- 目標測試覆蓋正常成功、conflict 後成功、非 conflict、retry exhaustion、reference change、opt-out 與 cancellation。
- mocks 必須記錄 GET/update 次數、resourceVersion、metadata 與 Data，不用 sleep 判斷 I/O 是否開始。
- retry 測試使用 instance-level 零延遲 backoff，避免修改 package global 或造成 flaky timing。
- 執行 `go test -race -count=1 ./...`、`go vet ./...`、`golangci-lint run ./...`。
- 執行 `gofmt -l`、`git diff --check` 與 Boundary 檢查。

## 已知風險

- 持續 conflict 仍會失敗；bounded retry 只降低短暫競爭的影響。
- reference 在 conflict 後改變時，本輪不立即重新 fetch，資料一致性延後到下一輪，以換取不跨引用寫入的安全性。
- 使用最新物件整體 Update 仍可能與其他 writer 競爭；每次 conflict 都重新 GET，且不覆蓋最新 metadata。
- client-go 預設 backoff 可能隨 dependency 升級改變；本 spec 不鎖死毫秒值，但固定使用目前 module 版本提供的 `DefaultBackoff`。

## 待確認項目

- 無。採 fail-closed reference revalidation、固定 client-go backoff 與不重新 fetch 的設計。
