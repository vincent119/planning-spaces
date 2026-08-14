# 需求文件：Secret 刪鍵與撤銷語意

## 來源

- Draft: 無
- Type: BugFix
- Owner: Vincent
- Status: Complete

## 文件定位

本 spec 接續 `.specs/2026-08-04-10-52_Feature-secure-secret-access/`，處理該 spec 明確列為非目標的 Secret 同步刪鍵語意。本次只修正 Sync Worker 對 `Secret.Data` 的對帳行為，並補齊撤銷、空集合與跨 namespace 授權測試；不重寫 Vault/AWS client、Webhook 注入、policy schema 或 Kubernetes repository。

參考來源：

- 需求來源: 使用者要求「修正 Secret 刪鍵語意，加入撤銷、空集合與跨 namespace 授權測試」
- 既有文件: `.specs/2026-08-04-10-52_Feature-secure-secret-access/requirements.md`、`docs/annotations.zh-Hant.md`
- 既有程式碼: `internal/syncer/application/sync_worker_usecase.go`、`internal/syncer/application/sync_worker_usecase_test.go`、`internal/syncer/domain/policy.go`

## 背景

Sync Worker 會從 Vault 或 AWS Secrets Manager 取得遠端 key-value 快照，再更新 Kubernetes Secret。目前實作只遍歷遠端回傳的 key，新增或覆寫同名 key，不會移除本地存在但遠端已刪除的 key。

這會讓已在機密後端撤銷的憑證繼續留在 Kubernetes Secret，造成撤銷不完整與最小權限失效。另一方面，必須區分「成功取得空集合」與「取得失敗」，避免後端故障被誤解為應清空 Secret。

## 問題陳述

遠端機密刪除 key 後，Sync Worker 不會從 Kubernetes Secret 移除對應 key，使舊機密持續可用。同步邏輯也尚未以測試固定空集合與跨 namespace 授權拒絕時的零副作用契約。

## 目標

1. 在遠端 fetch 成功後，將回傳快照視為受管 Kubernetes `Secret.Data` 的完整期望狀態。
2. 遠端已刪除的 key 必須在下次成功同步時從 Kubernetes Secret 撤銷。
3. 成功回傳空 map 時，清空非空 `Secret.Data`；當前已為空時不發出無效 Update。
4. fetch 失敗、未知 backend 或授權拒絕時，不得刪除或改寫任何現有資料。
5. 用測試固定 Secret namespace 與 backend path namespace 不一致時，在 fetch 前拒絕且不呼叫 Update。

## 非目標

1. 不刪除 Kubernetes Secret 物件本身，只對帳 `Secret.Data`。
2. 不新增 merge、preserve keys 或欄位所有權設定；受管 Secret 仍視為 Sync Worker 專屬管理。
3. 不改變 Vault/AWS `FetchSecret` 對找不到 path/key 的錯誤契約。
4. 不修正 Worker shutdown、retry、conflict retry、cache 或 singleflight。
5. 不改變 Webhook JSON Patch 與 Pod 注入行為。
6. 不調整 policy YAML schema、RBAC 或 deployment manifests。

## 已定決策

- 成功 fetch 的 `map[string]string` 是整個 `Secret.Data` 的權威快照，不與舊資料 merge。
- `SecretRef.Keys` 有指定 key 時，fetcher 回傳的選定快照仍會取代整個 `Secret.Data`；受管 Secret 不保留未被選定的手動 key。
- 成功回傳非 nil 或 nil 的空 map 都代表期望狀態為空；只有非 nil error 代表 fetch 失敗。
- 必須先完成 sync authorization，再 fetch 與計算差異。
- 只有快照內容不同時呼叫 `UpdateSecret`；map 順序、nil map 與空 map 不應造成重複 Update。
- 不記錄機密值，日誌只可包含 namespace、Secret name、backend 與必要的非敏感計數。

## 待確認項目

- 無。本 spec 依既有文件「覆寫 `Secret.Data`」與使用者要求，將受管 Secret 定義為完整快照同步。

## 現有行為

1. fetch 成功後，只把回傳 map 的 key 寫入 `Secret.Data`。
2. 現有 `Secret.Data` 中未出現於遠端回傳的 key 會被保留。
3. 遠端回傳空 map 時，Worker 不會更新非空 Secret。
4. 已有未授權測試會斷言 fetch/update 次數為零，但測試名稱與情境未明確固定跨 namespace 契約與原資料不變。

## 新行為

1. fetch 成功後，Worker 先建立全新 `map[string][]byte` 期望快照。
2. Worker 以內容比較現有與期望快照；相同時不 Update，不同時以期望快照整體取代 `Secret.Data`。
3. 遠端缺少的舊 key 會被刪除；遠端為空時會清空現有資料。
4. 任何 fetch error 都終止本次 Secret 對帳，不建立空快照、不 Update。
5. Sync policy 以 `secret.Namespace` 對 `{{namespace}}` 展開；`team-a` Secret 引用 `teams/team-b/...` 時拒絕，不 fetch、不 Update、不改變原 Data。

## 影響範圍

- 使用者: 使用 Sync Worker 管理 Kubernetes Secret 的平台與應用團隊
- 功能: 背景 Secret 同步、機密撤銷、sync workload authorization 回歸驗證
- API / CLI: 無新增介面；既有 annotation 契約不變
- Data / Storage: Kubernetes `Secret.Data` 從 merge 語意改為完整快照取代
- 文件 / 安裝 / 發布: 需更新繁體中文 annotation 文件；無設定與安裝變更

## 使用情境

- 作為平台管理者，我想要遠端已撤銷的 key 自動從 Kubernetes Secret 移除，以便撤銷能在每個密鑰副本生效。
- 作為安全工程師，我想要成功空回應與後端錯誤具有不同結果，以便故障不會意外刪除機密。
- 作為多租戶叢集管理者，我想要明確驗證跨 namespace path 在 fetch 前被拒絕，以便租戶無法讀取或影響其他 namespace 的機密。

## 驗收情境

### 情境：遠端刪鍵會撤銷本地 key

- 場景：現有 Secret 有保留 key 與已撤銷 key
- 測試：`TestSyncOnce_ReplacesDataAndRevokesMissingKeys`
- 假設：Kubernetes `Secret.Data` 含 `ACTIVE=newer-local`與 `REVOKED=old`，fetch 成功只回傳 `ACTIVE=current`
- 當：執行一次 `syncOnce`
- 那麼：只呼叫一次 Update，結果只含 `ACTIVE=current`，`REVOKED` 不存在

### 情境：成功空集合清空 Secret.Data

- 場景：遠端 path 存在且 fetch 成功，但回傳空 map
- 測試：`TestSyncOnce_EmptyRemoteDataClearsSecret`
- 假設：現有 `Secret.Data` 至少有一個 key
- 當：執行一次 `syncOnce`
- 那麼：只呼叫一次 Update，更新後 `Secret.Data` 長度為零

### 情境：已空快照不重複更新

- 場景：遠端與 Kubernetes Secret 都是空集合
- 測試：`TestSyncOnce_EmptySnapshotsSkipUpdate`
- 假設：現有 `Secret.Data` 為 nil 或空 map，fetch 成功回傳 nil 或空 map
- 當：執行一次 `syncOnce`
- 那麼：不呼叫 Update

### 情境：fetch 失敗不得誤刪資料

- 場景：遠端讀取失敗
- 測試：`TestSyncOnce_FetchFailurePreservesData`
- 假設：現有 Secret 含機密資料，fetcher 回傳非 nil error
- 當：執行一次 `syncOnce`
- 那麼：不呼叫 Update，原 `Secret.Data` 內容不變

### 情境：指定 keys 仍使用完整快照取代

- 場景：annotation 只指定一個 key，現有 Secret 含其他手動 key
- 測試：`TestSyncOnce_RequestedKeysReplaceWholeDataSnapshot`
- 假設：fetcher 成功回傳選定 key 的 map
- 當：執行一次 `syncOnce`
- 那麼：更新後只保留 fetcher 回傳的 key，不保留手動 key

### 情境：跨 namespace 引用在 fetch 前拒絕

- 場景：`team-a` Secret 引用 `teams/team-b/database`
- 測試：`TestSyncWorker_CrossNamespaceReferenceDoesNotFetchOrUpdate`
- 假設：sync policy 只允許 `teams/{{namespace}}/`，且現有 Secret 含舊資料
- 當：執行一次 `syncOnce`
- 那麼：fetch 次數與 Update 次數都為零，原 Data 不變，授權理由不包含敏感 path

### 情境：既有行為不被破壞

- 場景：同步新值、相同快照、未設 path、未知 backend 與優雅關機
- 測試：`TestSyncOnce_UpdatesWhenValueDiffers|TestSyncOnce_SkipsWhenValuesIdentical|TestSyncOnce_SkipsSecretWithoutPath|TestSyncOnce_UnknownBackendIsSkipped|TestRun_StopsOnContextCancellation`
- 假設：使用既有測試資料與 mock
- 當：執行 application package 測試
- 那麼：既有成功、skip 與 shutdown 結果維持不變

## 驗收條件

1. 「遠端刪鍵會撤銷本地 key」與「指定 keys 仍使用完整快照取代」測試通過。
2. 「成功空集合清空 Secret.Data」與「已空快照不重複更新」同時通過。
3. 「fetch 失敗不得誤刪資料」測試明確斷言 Update 為零且原 Data 不變。
4. 「跨 namespace 引用在 fetch 前拒絕」測試明確斷言 fetch/update 為零且原 Data 不變。
5. `go test -race -count=1 ./...`、`go vet ./...`、`golangci-lint run ./...` 與 `git diff --check` 全部通過。
6. `docs/annotations.zh-Hant.md` 明確說明完整快照、刪鍵、空集合與 fetch error 保留語意。

## 驗證需求

- Unit / Integration: `go test ./internal/syncer/application -run 'TestSync(Once|Worker)_(ReplacesDataAndRevokesMissingKeys|EmptyRemoteDataClearsSecret|EmptySnapshotsSkipUpdate|FetchFailurePreservesData|RequestedKeysReplaceWholeDataSnapshot|CrossNamespaceReferenceDoesNotFetchOrUpdate)'`
- CLI / Dry-run: 無
- 文件檢查: 確認 `docs/annotations.zh-Hant.md` 與 application 對帳邏輯一致
- 回歸驗證: `go test -race -count=1 ./...`、`go vet ./...`、`golangci-lint run ./...`

## 風險與假設

| 類型 | 內容 | 處理方式 |
|------|------|----------|
| 風險 | 完整快照取代會刪除手動加入的 key | 文件明確宣告受管 Secret 為 Worker 專屬管理，並加入指定 keys 測試 |
| 風險 | 將 fetch error 誤當空回應會刪除全部機密 | 只在 `err == nil` 後建立期望快照，專項測試錯誤路徑 |
| 風險 | map 順序或 nil/空差異造成 Update 風暴 | 使用內容相等比較，將 nil 與空 map 視為等價 |
| 風險 | 跨 namespace 拒絕只測決策，未測副作用 | application 測試同時斷言 fetch、Update 與原 Data |
| 假設 | 帶同步 label 的 Secret 不由其他 controller 或人工共同管理 Data | 寫入繁體中文 annotation 文件，不在本次新增 merge 模式 |

## 摘要

- 關鍵決策: 成功 fetch 為完整快照取代；空快照代表全部撤銷；fetch error 保留原資料
- 待確認項目: 無
- 風險: 手動 key 將被完整快照移除，必須在文件明確宣告所有權
- 下一步: 實作與驗證已完成，可進入 Git 提交與審查
