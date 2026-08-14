# 任務文件：Secret 刪鍵與撤銷語意

Status: Complete

## Execution Context

- 意圖: 將 Sync Worker 從只新增/覆寫 key 的 merge 行為，修正為以成功 fetch 結果整體對帳 `Secret.Data`，使遠端刪鍵與空集合能撤銷 Kubernetes 內的舊資料，並補齊跨 namespace 授權的零副作用測試。
- 非目標: 不刪除 Secret 物件；不新增 merge/preserve 開關；不改 Vault/AWS fetcher、policy schema、RBAC、Webhook、deployment、Worker shutdown 或 retry。
- 已定決策: 成功 fetch 是完整權威快照；空 map 會清空非空 Data；fetch error 保留原 Data；指定 keys 也不保留快照外的手動 key；authorization 必須在 fetch 與差異計算前完成。
- 邊界: 主要變更限於 `internal/syncer/application/sync_worker_usecase*.go`；可在 `internal/syncer/domain/policy_test.go` 補 sync 跨 namespace 決策測試，並更新 `docs/annotations.zh-Hant.md` 與本 spec。
- 關鍵檔案: `internal/syncer/application/sync_worker_usecase.go`、`internal/syncer/application/sync_worker_usecase_test.go`、`internal/syncer/domain/policy_test.go`、`docs/annotations.zh-Hant.md`
- 完成條件: 撤銷、空集合、錯誤保留、指定 keys 與跨 namespace 測試通過；全套 race test、vet、lint、格式與 diff 檢查通過；文件與實作一致。

### Protected Behavior

- authorization deny 必須在 fetch 前返回，不 fetch、不 Update、不改寫 in-memory Secret Data。
- fetch error 不得被解讀為空快照，不得刪除現有 key。
- 未設 path、未知 backend 繼續 skip，原 Data 不變。
- 相同快照繼續不 Update，新增或修改 value 繼續正常更新。
- Webhook mutation、SecretFetcher、Policy loader、K8sRepository、RBAC 與 deployment 契約不變。
- 日誌、metrics 與 error 不得包含機密值。

### 邊界

#### Allowed Changes

- `internal/syncer/application/sync_worker_usecase.go`
- `internal/syncer/application/sync_worker_usecase_test.go`
- `internal/syncer/domain/policy_test.go`
- `docs/annotations.zh-Hant.md`
- `.specs/2026-08-04-11-43_BugFix-secret-key-revocation-semantics/`

#### Forbidden

- 不修改 `internal/syncer/infra/vault_client.go`、`aws_client.go` 或 `k8s_repository.go`。
- 不修改 `internal/syncer/domain/policy.go`、policy YAML schema 或 authorization default deny 行為。
- 不修改 Webhook、configuration、metrics、RBAC、deployment manifests 與 listener。
- 不刪除 Kubernetes Secret 物件，不引入新相依，不新增行為開關。
- 不為通過測試而改變 fetch error、authorization 或 unknown backend 早退契約。
- 不將 `_workspace/` 納入提交。

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 以測試固定快照、撤銷與空集合契約 | 無 | Complete | 紅燈已確認現有 merge bug |
| T2 實作完整快照取代 | T1 | Complete | 快照對帳與 application race test 通過 |
| T3 補強跨 namespace sync 授權測試 | T1 | Complete | domain 決策與 application 零副作用通過 |
| T4 更新撤銷與所有權文件 | T2 | Complete | 已說明完整快照與專屬所有權 |
| V1 驗收情境覆蓋 | T2、T3、T4 | Complete | 六個主要驗收情境與 domain 對照通過 |
| V2 回歸驗證 | V1 | Complete | 全套 race test 與 go vet 通過 |
| V3 品質檢查清單 | V1、V2 | Complete | lint、格式、Boundary 與 diff 檢查通過 |

## 實作任務

- [x] T1 以測試固定快照、撤銷與空集合契約
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/application/sync_worker_usecase_test.go`
    - Forbidden: 修改 production code、放寬現有測試斷言、只檢查 Update 次數而不檢查結果 Data
  - Depends: 無
  - Context: 目前邏輯會保留遠端已缺少的 key。先加入撤銷、空集合、空幂等、fetch error 保留與指定 keys 完整取代測試，確認測試能捕捉現有 bug。
  - Verify:
    - `go test ./internal/syncer/application -run 'TestSyncOnce_(ReplacesDataAndRevokesMissingKeys|EmptyRemoteDataClearsSecret|EmptySnapshotsSkipUpdate|FetchFailurePreservesData|RequestedKeysReplaceWholeDataSnapshot)'`
    - 在 T2 前，撤銷與非空轉空測試必須因現有 merge 行為而失敗；記錄失敗原因到 Implementation Notes。
    - fetch error 測試斷言 fetch 一次、Update 零次、原 Data 與測試前 DeepCopy 相同。

- [x] T2 實作完整快照取代
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/application/sync_worker_usecase.go`、必要的 `sync_worker_usecase_test.go` 測試輔助調整
    - Forbidden: 改動 `SecretFetcher`、`K8sSecretRepo`、authorization 介面與早退順序；新增外部 dependency；記錄機密值
  - Depends: T1
  - Context: fetch 成功後建立新 desired `map[string][]byte`，以內容比較 current/desired。非空轉空是 changed，nil/空都是零 key 且應幂等。只有 changed 才整體指派 `secret.Data` 並 Update。
  - Verify:
    - `go test ./internal/syncer/application -run 'TestSyncOnce_(ReplacesDataAndRevokesMissingKeys|EmptyRemoteDataClearsSecret|EmptySnapshotsSkipUpdate|FetchFailurePreservesData|RequestedKeysReplaceWholeDataSnapshot|UpdatesWhenValueDiffers|SkipsWhenValuesIdentical)'`
    - `go test -race -count=1 ./internal/syncer/application`
    - 檢查實作在 `fetchErr != nil` 分支後才建立 desired map，且不在比較前修改 current Data。

- [x] T3 補強跨 namespace sync 授權測試
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/application/sync_worker_usecase_test.go`、`internal/syncer/domain/policy_test.go`
    - Forbidden: 修改 `policy.go`、放寬 namespace/path template matcher、將跨 namespace 錯誤移到 fetch 後
  - Depends: T1
  - Context: 使用 `namespace_globs: team-*` 與 `teams/{{namespace}}/` sync rule。Domain 測試固定 `team-a` 只可用 `teams/team-a/...`；application 測試需保留原 Secret DeepCopy，斷言 deny 後 fetch/update 為零與 Data 不變。
  - Verify:
    - `go test ./internal/syncer/domain -run 'TestAuthorizer_SyncPathTemplateRejectsCrossNamespace'`
    - `go test ./internal/syncer/application -run 'TestSyncWorker_CrossNamespaceReferenceDoesNotFetchOrUpdate'`
    - 測試同時包含同 namespace allow 的對照，避免因 policy 完全無法匹配而得到假陽性。

- [x] T4 更新撤銷與所有權文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: `docs/annotations.zh-Hant.md`、本 spec 的 Implementation Notes
    - Forbidden: 新增未實作的設定、merge 模式、retry 保證或 Kubernetes Secret 物件刪除說明
  - Depends: T2
  - Context: 在 Secret sync 段落明確說明遠端成功快照取代整個 Data、缺少 key 會撤銷、空快照會清空、fetch error 保留原 Data，並警示不可混用手動 key。
  - Verify:
    - `rg -n '完整快照|撤銷|空集合|fetch.*失敗|專屬' docs/annotations.zh-Hant.md`
    - 人工比對文件與 `sync_worker_usecase.go` 的實際早退與取代行為。

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Status: Complete
  - Boundary:
    - Allowed Changes: 任務邊界內測試、本 spec Implementation Notes
    - Forbidden: 移除或弱化 requirements 中任一副作用斷言
  - Depends: T2、T3、T4
  - Context: 確認 requirements 的六個主要驗收情境都有可執行測試，測試 selector 與文件名稱一致。
  - Verify:
    - `go test ./internal/syncer/application -run 'TestSync(Once|Worker)_(ReplacesDataAndRevokesMissingKeys|EmptyRemoteDataClearsSecret|EmptySnapshotsSkipUpdate|FetchFailurePreservesData|RequestedKeysReplaceWholeDataSnapshot|CrossNamespaceReferenceDoesNotFetchOrUpdate)'`
    - `go test ./internal/syncer/domain -run 'TestAuthorizer_SyncPathTemplateRejectsCrossNamespace'`
    - 逐項對照 `requirements.md` 驗收情境的 fetch 次數、Update 次數、結果 Data 與原 Data 斷言。

- [x] V2 回歸驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: 僅修正由本 BugFix 直接造成的測試失敗，並先更新 Implementation Notes
    - Forbidden: 重寫 Webhook、fetcher、policy loader、repository 或 Protected Behavior
  - Depends: V1
  - Context: 全套測試確認新快照語意沒有破壞 Webhook、authorization、Vault/AWS 與 Worker shutdown。
  - Verify:
    - `go test -race -count=1 ./...`
    - `go vet ./...`
    - 既有 `TestSyncOnce_UpdatesWhenValueDiffers`、`TestSyncOnce_SkipsWhenValuesIdentical`、`TestSyncOnce_SkipsSecretWithoutPath`、`TestSyncOnce_UnknownBackendIsSkipped`、`TestRun_StopsOnContextCancellation` 全部通過。

- [x] V3 品質檢查清單
  - Status: Complete
  - Boundary:
    - Allowed Changes: 格式修正、任務邊界內測試與文件、Implementation Notes
    - Forbidden: 為通過 lint 而放寬刪鍵、錯誤保留或授權契約
  - Depends: V1、V2
  - Context: 執行最終格式、lint、diff 與文件一致性檢查，確認變更未超出 Execution Context。
  - Verify:
    - `gofmt -l internal/syncer/application internal/syncer/domain` 無輸出。
    - `golangci-lint run ./...`
    - `git diff --stat`、`git diff --check`
    - `git diff --name-only` 只包含 Allowed Changes，且 `_workspace/` 未納入追蹤。
  - 品質項目:
    - [x] 遠端缺少 key 會從 Kubernetes `Secret.Data` 撤銷
    - [x] 成功空 map 會清空非空 Data
    - [x] nil/空快照相等時不 Update
    - [x] fetch error 不 Update 且原 Data 不變
    - [x] 指定 keys 時仍以回傳快照取代整個 Data
    - [x] 跨 namespace path 在 fetch 前拒絕
    - [x] 跨 namespace deny 後 fetch/update 為零且原 Data 不變
    - [x] 未設 path 與未知 backend 不刪除資料
    - [x] 相同快照不產生無效 Update
    - [x] 日誌、metric 與 error 無機密值
    - [x] 文件明確宣告受管 Secret 專屬所有權
    - [x] 主要驗收情境已覆蓋
    - [x] Protected Behavior 回歸驗證通過
    - [x] `git diff --check` 通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的 `Execution Context`
2. 目前未完成 task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個 `.specs` 目錄。若文件很大，先用標題與關鍵字定位：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-11-43_BugFix-secret-key-revocation-semantics
```

## Implementation Notes

- 2026-08-04: 已建立規劃，尚未執行 production code 與測試變更。
- T1 完成：新增撤銷、空集合、空幂等、fetch error 保留與指定 keys 測試。實作前 selector 如預期失敗：現有 merge 保留 `REVOKED`/`MANUAL`，且成功空 map 不會 Update。
- T2 完成：在 fetch 成功後建立新 desired map，以 key 數、存在性與 byte 內容比較，只在有差異時整體取代 `Secret.Data`；指定測試與 application race test 通過。
- T3 完成：sync path template 的同 namespace allow 與跨 namespace deny 皆通過；deny 路徑的 fetch/update 為零且原 Data 不變。
- T4 完成：`docs/annotations.zh-Hant.md` 已說明完整快照、撤銷、空集合、fetch error 保留與專屬所有權。
- V1 完成：application 六個驗收情境與 domain 跨 namespace 決策測試通過。
- V2 完成：`go test -race -count=1 ./...` 與 `go vet ./...` 通過。
- V3 完成：`golangci-lint run ./...` 回報 `0 issues`，gofmt、Boundary、文件關鍵字與 `git diff --check` 通過；`_workspace/` 仍為未追蹤產物。
- 現有 bug 位於 `sync_worker_usecase.go` fetch 成功後的 merge loop；該 loop 不會刪除 fetched map 缺少的 current key。
- 現有 `TestSyncWorker_UnauthorizedReferenceDoesNotFetchOrUpdate` 已檢查一組 namespace/path 不一致情境，T3 應將契約具體化為 path template 測試，並補原 Data 不變與同 namespace allow 對照。

## 驗證結果摘要

- 新行為驗證: 通過；撤銷、空集合、空幂等、fetch error 保留、指定 keys 與跨 namespace 測試成功
- 回歸驗證: 通過；`go test -race -count=1 ./...`、`go vet ./...`、`golangci-lint run ./...`
- 文件一致性: 已確認 `docs/annotations.zh-Hant.md` 與完整快照實作一致
- 剩餘風險: 完整快照取代會移除手動 key，已透過文件警示與指定測試固定所有權

## 後續改善

- [ ] 另案評估 field ownership 或 preserve keys 模式，不納入本 BugFix。
- [ ] 另案評估 Kubernetes Update conflict retry 與 resourceVersion 衝突指標。
- [ ] 另案處理 Sync Worker shutdown 進行中操作的 context 契約。
