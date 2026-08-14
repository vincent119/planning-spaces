# 任務文件：Kubernetes Graceful Termination

Status: Complete

## Execution Context

- 意圖: 將 Pod termination budget 與 5 秒 preStop、30 秒 application cleanup 對齊，並讓 `/readyz` 在 signal/server error exit path 轉為 `503`。
- 非目標: 不改 graceful timeout、Worker/HTTP/tracer cleanup、config、image、PDB、replicas、Webhook 或 Secret sync。
- 已定決策: native preStop sleep 5 秒；termination grace 45 秒；atomic readiness 初始 true 且 shutdown 單向 false；signal返回 nil，server error保留原 error；不新增 application sleep。
- 邊界: 變更限於 main readiness/lifecycle、base Deployment、README、繁體中文 deployment文件與本 spec。
- 關鍵檔案: `cmd/vault-agent/main.go`、`cmd/vault-agent/main_test.go`、`deployments/kustomize/base/deployment.yaml`、`README.md`、`docs/deploy.zh-Hant.md`
- 完成條件: readiness、signal、server error unit tests 通過；base/prod render 包含 45/5 且無 exec sleep；全套 race、vet、lint、格式與 diff 檢查通過。

### Protected Behavior

- `/healthz` 在 process 可服務期間仍回傳 `200`。
- `/readyz` 正常服務時仍回傳 `200`。
- signal task path仍返回 nil，server error path仍返回原 error。
- Worker 由 task context取消，cleanup bounded wait 不變。
- graceful 30 秒 shared timeout 與 HTTP、Worker、tracer LIFO 順序不變。
- Chainguard static image、non-root security context 與現有 probes 不變。
- runtime log/HTTP response 使用英文，不記錄機密值。

### 邊界

#### Allowed Changes

- `cmd/vault-agent/main.go`
- `cmd/vault-agent/main_test.go`
- `deployments/kustomize/base/deployment.yaml`
- `README.md`
- `docs/deploy.zh-Hant.md`
- `.specs/2026-08-04-14-39_BugFix-kubernetes-graceful-termination/`

#### Forbidden

- 不修改 `commons/graceful`、Sync Worker、Vault/AWS client、Webhook、authorization、metrics schema 或 config schema。
- 不修改 Dockerfile、base image、PDB、replica count、service、RBAC 或 admission manifests。
- 不改 `graceful.WithTimeout(30 * time.Second)` 或 cleanup option order。
- 不加入 exec/shell preStop，不新增 dependency 或 application sleep。
- 不宣稱本地 render 等同 target cluster API 相容性驗證。
- 不將 `_workspace/` 納入提交。

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 以 red tests 固定 readiness/lifecycle 契約 | 無 | Complete | handler 與兩個 exit path |
| T2 實作 atomic readiness 與 wait helper | T1 | Complete | 不改 cleanup options |
| T3 對齊 Deployment termination budget | T2 | Complete | native sleep，不用 exec |
| T4 更新操作與功能文件 | T3 | Complete | 記錄 5/30/45 時間軸 |
| V1 驗收情境覆蓋 | T2、T3、T4 | Complete | requirements A-E |
| V2 回歸驗證 | V1 | Complete | Worker、HTTP、Webhook、安全契約 |
| V3 品質與 manifest檢查 | V1、V2 | Complete | race、vet、lint、render、Boundary |

## 實作任務

- [x] T1 以 red tests 固定 readiness/lifecycle 契約
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/main_test.go`
    - Forbidden: 修改 production code；送真實 OS signal；啟動 network listener；用 sleep 同步
  - Depends: 無
  - Context: 新增 readiness handler 狀態轉換、已取消 context path、buffered server error path測試。signal/error tests 必須斷言 mark false，error path另斷言 errors.Is 保留 cause。
  - Verify:
    - `go test ./cmd/vault-agent -run 'Test(ReadinessHandler_TransitionsToNotReady|WaitForTermination_ContextMarksNotReady|WaitForTermination_ServerErrorMarksNotReady)'`
    - T2 前應因 helper 尚不存在而 compile fail，證據記錄至 Implementation Notes。
    - `git diff --stat`、`git diff --check`

- [x] T2 實作 atomic readiness 與 wait helper
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/main.go`、必要的 `main_test.go` 輔助調整
    - Forbidden: 改 HTTP server timeout/route ownership、cleanup options、Worker helper、graceful timeout；使用未同步 bool
  - Depends: T1
  - Context: 以 package-private `readinessState` 實作 `http.Handler`，初始 true、shutdown 單向 false。以 `waitForTermination` 集中 signal/server error select，兩個 branch都先 mark false；main 的 `/readyz` 改用該 handler。
  - Verify:
    - `go test ./cmd/vault-agent -run 'Test(ReadinessHandler_TransitionsToNotReady|WaitForTermination_ContextMarksNotReady|WaitForTermination_ServerErrorMarksNotReady|StartSyncWorker_CancelAndWait|WaitForSyncWorker_DeadlineExceeded)'`
    - `go test -race -count=1 ./cmd/vault-agent`
    - 程式碼檢查 `/healthz` 仍固定 200，`graceful.WithTimeout(30*time.Second)` 與 cleaner order不變。
    - `git diff --stat`、`git diff --check`

- [x] T3 對齊 Deployment termination budget
  - Status: Complete
  - Boundary:
    - Allowed Changes: `deployments/kustomize/base/deployment.yaml`
    - Forbidden: 修改 image/securityContext/probes/resources/env/volume/RBAC/service/PDB；加入 exec hook
  - Depends: T2
  - Context: Pod spec設定 `terminationGracePeriodSeconds: 45`；vault-agent container設定 native `lifecycle.preStop.sleep.seconds: 5`。prod overlay不重複 patch，直接繼承 base。
  - Verify:
    - `kubectl kustomize deployments/kustomize/base`
    - `kubectl kustomize deployments/kustomize/overlays/prod`
    - render 結果各自只有一個 `terminationGracePeriodSeconds: 45`、一個 `preStop`、一個 `sleep` 與 `seconds: 5`。
    - `rg -n 'exec:|command:.*sleep' deployments/kustomize/base/deployment.yaml` 無輸出。
    - `git diff --stat`、`git diff --check`

- [x] T4 更新操作與功能文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: `README.md`、`docs/deploy.zh-Hant.md`、本 spec Implementation Notes
    - Forbidden: 宣稱零中斷、宣稱本地 render 已驗證 target cluster；修改英文部署文件
  - Depends: T3
  - Context: README 更新優雅關機功能列；繁中部署文件記錄 5/30/45、native sleep/static image 原因、target cluster dry-run 與 rollback 注意事項。
  - Verify:
    - `rg -n '45 秒|30 秒|5 秒|preStop|readyz|503|dry-run' README.md docs/deploy.zh-Hant.md`
    - 人工比對文件、main 與 rendered manifest。

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Status: Complete
  - Boundary:
    - Allowed Changes: 任務邊界內測試、manifest、文件、本 spec Implementation Notes
    - Forbidden: 以 sleep、真實 signal 或 live cluster取代 deterministic tests
  - Depends: T2、T3、T4
  - Context: 對照 requirements A-E，確認 handler state、兩個 exit path、manifest budget 與既有 Worker lifecycle都有可執行驗證。
  - Verify:
    - `go test ./cmd/vault-agent -run 'Test(ReadinessHandler_TransitionsToNotReady|WaitForTermination_ContextMarksNotReady|WaitForTermination_ServerErrorMarksNotReady|StartSyncWorker_CancelAndWait|WaitForSyncWorker_DeadlineExceeded)'`
    - base/prod Kustomize render 與欄位計數檢查通過。

- [x] V2 回歸驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: 僅修正本 BugFix 直接造成的邊界內失敗，並先回填 Implementation Notes
    - Forbidden: 重寫 Worker、HTTP cleanup、Webhook、authorization、Secret sync 或 graceful library
  - Depends: V1
  - Context: 全套測試確認 readiness orchestration 沒有破壞 server、Worker 與 admission行為。
  - Verify:
    - `go test -race -count=1 ./...`
    - `go vet ./...`
    - 既有 `TestStartSyncWorker_CancelAndWait`、Webhook handler tests、Sync Worker cancellation tests 通過。

- [x] V3 品質與 manifest 檢查
  - Status: Complete
  - Boundary:
    - Allowed Changes: 格式修正、邊界內測試/manifest/文件、Implementation Notes
    - Forbidden: 為通過 lint/render 而移除 atomic state、native sleep、budget 或 exit error 斷言
  - Depends: V1、V2
  - Context: 執行 race、vet、lint、格式、Kustomize、Boundary 與 diff檢查，確認外部未追蹤產物未納入。
  - Verify:
    - `gofmt -l cmd/vault-agent` 無輸出。
    - `golangci-lint run ./...`
    - `kubectl kustomize deployments/kustomize/base` 與 prod overlay成功。
    - `git diff --stat`、`git diff --check`
    - `git status --short` 確認 `_workspace/` 仍未追蹤。
  - 品質項目:
    - [x] readiness 初始 200
    - [x] signal path readiness 503
    - [x] server error path readiness 503且保留 cause
    - [x] readiness state 無 data race
    - [x] healthz 維持 200
    - [x] Worker cancellation/wait不變
    - [x] graceful cleanup timeout維持 30 秒
    - [x] cleaner order維持 HTTP、Worker、tracer
    - [x] terminationGracePeriodSeconds 為 45
    - [x] native preStop sleep 為 5 秒
    - [x] manifest 無 exec/shell sleep
    - [x] base/prod render一致
    - [x] 文件清楚記錄 5/30/45與 target cluster dry-run
    - [x] target cluster server-side dry-run 接受 native lifecycle sleep
    - [x] runtime訊息為英文且無機密值
    - [x] `git diff --check` 通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的 `Execution Context`
2. 目前未完成 task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個 `.specs` 目錄。若文件很大，先用標題與關鍵字定位：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-14-39_BugFix-kubernetes-graceful-termination
```

## Implementation Notes

- 2026-08-04: 已建立規劃，尚未修改 production code、tests、manifest 或 user-facing docs。
- 2026-08-04: T1 red test 在 compile 階段確認 `newReadinessState` 與 `waitForTermination` 尚不存在；三個目標 selector 均按預期失敗。
- 2026-08-04: T2 atomic readiness、signal/server error wait helper 與 `/readyz` handler 已完成；目標測試與 main package race test 通過，既有 Worker lifecycle tests 無回歸。
- 2026-08-04: T3 base Deployment 已設定 native preStop sleep 5 秒與 termination grace 45 秒；base/prod Kustomize render 均成功且只出現一組 5/45 contract，無 exec hook。
- 2026-08-04: T4 README 與繁中部署文件已記錄 5/30/45 時間軸、`/readyz` 503、static image 原因、target cluster server-side dry-run 與 rollback 注意事項。
- 2026-08-04: V1 requirements A-E 全數通過；main tests 與 base/prod manifest field counts 符合預期。
- 2026-08-04: V2 `go test -race -count=1 ./...` 與 `go vet ./...` 通過，Worker、Webhook、Secret sync 與既有 cleanup 無回歸。
- 2026-08-04: V3 `golangci-lint run ./...` 回報 0 issues；格式、Kustomize render、Boundary 與 diff 檢查通過。
- 2026-08-04: 使用者在目標 cluster 執行 `kubectl apply --dry-run=server -k deployments/kustomize/overlays/prod`，所有資源均通過，Deployment API 接受 native lifecycle sleep。
- application cleanup budget 固定 30 秒；base Deployment 已設定 termination grace 45 秒。
- container 使用 Chainguard static image，不可依賴 exec `sleep`；本機 render 與目標 cluster server-side dry-run 均支援 native sleep action。
- prod overlay 直接繼承 base Deployment，已確認無須新增 overlay patch。

## 驗證結果摘要

- 新行為驗證: readiness 與兩個 lifecycle exit path 測試全部通過
- 回歸驗證: 全套 race test、vet 與 lint 通過
- manifest驗證: base/prod render 成功，各只有一組 termination 45、native sleep 5，無 exec；目標 cluster server-side dry-run 通過
- 文件一致性: README、繁中部署文件、requirements/design/tasks 與 5/30/45 實作一致
- 剩餘風險: 尚未以實際 rollout 量測 preStop、SIGTERM、request drain 與 cleanup 的完整時間軸

## 後續改善

- [ ] 另案評估 readiness 初始 false 並在兩個 listeners bind後才轉 true。
- [ ] 另案評估 target cluster end-to-end termination測試與 request drain量測。
- [ ] 另案評估 Worker error-returning `errgroup` lifecycle。
