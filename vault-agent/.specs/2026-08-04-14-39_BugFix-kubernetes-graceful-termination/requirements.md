# 需求文件：Kubernetes Graceful Termination

## 來源

- Draft: 無
- Type: BugFix
- Owner: Vincent
- Status: Complete

## 文件定位

本 spec 接續 `.specs/2026-08-04-13-50_BugFix-sync-worker-graceful-shutdown/`，完成該文件留下的 Kubernetes termination budget 對齊工作，並修正 `/readyz` 在 application shutdown 期間仍固定回傳 `200` 的問題。本次保留既有 HTTP、Worker、tracer cleanup 順序，不重寫 `commons/graceful`、Sync Worker、Webhook、authorization 或 backend clients。

參考來源：

- code review 後續項目: deployment `terminationGracePeriodSeconds` 與 30 秒 application cleanup budget 對齊
- 既有 application: `cmd/vault-agent/main.go` 使用 `graceful.WithTimeout(30 * time.Second)`
- 既有 deployment: `deployments/kustomize/base/deployment.yaml` 未設定 termination budget 或 preStop
- 既有 readiness: `/readyz` 無狀態，固定回傳 `200`
- container contract: Chainguard static runtime 無 shell 或外部 `sleep` binary
- Kubernetes contract: 目前 API 與 `kubectl v1.33.2` 支援原生 `lifecycle.preStop.sleep`

## 背景

Kubernetes 未明確設定 `terminationGracePeriodSeconds` 時，Pod 使用平台預設值。application cleanup 本身最多可使用 30 秒，因此 Pod budget 若同樣只有 30 秒，就沒有時間容納 Endpoint propagation、preStop 或 runtime 收尾，最壞情況可能在 cleanup 即將完成時收到強制終止。

另一方面，operations server 的 `/readyz` 永遠回傳 `200`。當 graceful task 收到 SIGTERM 或任一 HTTP server 非預期停止時，process 已經準備退出，但 readiness contract 沒有反映狀態變更。

## 現有行為

- Deployment 未設定 `terminationGracePeriodSeconds`。
- Deployment 沒有 preStop lifecycle hook。
- `/readyz` 從 process 啟動到 operations server 關閉前都回傳 `200`。
- task 收到 signal 後直接返回，graceful manager 建立獨立 30 秒 cleanup context。
- cleanup 依序停止 HTTP、等待 Worker、關閉 tracer。
- container image 無 shell，不能使用 `exec.command: ["sleep", "5"]`。

## 新行為

- Pod 明確設定 `terminationGracePeriodSeconds: 45`。
- container 使用 Kubernetes 原生 `lifecycle.preStop.sleep.seconds: 5`，不依賴 shell 或 image 內工具。
- Kubernetes termination 時間軸為 5 秒 preStop drain、最多 30 秒 application cleanup、10 秒剩餘 buffer。
- `/readyz` 使用 concurrency-safe readiness state；正常服務時回傳 `200`，shutdown 或 server error path 開始退出時回傳 `503`。
- `/healthz` 在 process 存活期間仍回傳 `200`，避免 shutdown drain 被 liveness restart 干擾。
- signal 與 server error 兩個 task exit path 都先標記 not-ready，再進入既有 cleanup。
- 不增加 application 內部 sleep；Endpoint propagation 等待由 native preStop 負責。

## 目標

- 讓 Pod termination budget 大於 application cleanup 上限並保有明確 buffer。
- 在 static image 上提供不依賴 shell 的 bounded preStop drain。
- 讓 readiness 正確反映 process 已進入 shutdown。
- 保留 Worker cancellation、HTTP drain、tracer cleanup 與 shared 30 秒 budget。
- 讓 base 與 prod overlay render 出一致的 termination contract。

## 非目標

- 不新增 shutdown timeout、preStop duration 或 readiness 的 config/env。
- 不改變 `graceful.WithTimeout(30 * time.Second)`。
- 不替換 `commons/graceful`、不導入 `errgroup`。
- 不修改 liveness endpoint、Webhook route、authentication、authorization 或 Secret sync。
- 不加入 exec preStop、shell、busybox 或額外 container binary。
- 不保證所有 ingress、load balancer 或 CNI 都在 5 秒內完成 propagation。
- 不建立完整 SIGTERM end-to-end cluster test。
- 不修改 PDB、replica count、rolling update strategy 或 admission `failurePolicy`。

## 使用情境

### 情境一：正常 Pod termination

Kubernetes 將 Pod 標記 terminating，執行 5 秒 native preStop sleep，之後送出 SIGTERM。application 將 readiness 切為 false，取消 Worker 並在 30 秒內完成 HTTP、Worker、tracer cleanup。

### 情境二：cleanup 使用完整 budget

HTTP request 或 backend cancellation 接近 30 秒才完成。Pod 仍有總計 45 秒 budget，不會在 application timeout 同一時間立刻被強制終止。

### 情境三：HTTP server 非預期失敗

task 從 server error channel 收到 error，立即將 readiness 設為 false並開始 cleanup，不等待只適用於 Kubernetes termination 的 preStop。

### 情境四：shutdown 期間的 probe

operations server 尚未完成 Shutdown 時，`/healthz` 仍回傳 `200`，`/readyz` 回傳 `503`。

## 驗收情境

### 場景 A：readiness 狀態轉換

- 測試: `TestReadinessHandler_TransitionsToNotReady`
- 假設: readiness state 初始為 ready
- 當: handler 在正常與 mark-not-ready 後分別收到 request
- 那麼: status 依序為 `200` 與 `503`

### 場景 B：signal path 標記 not-ready

- 測試: `TestWaitForTermination_ContextMarksNotReady`
- 假設: task context 被取消
- 當: lifecycle wait helper 收到 `ctx.Done()`
- 那麼: helper 返回 nil，readiness 變成 false

### 場景 C：server error path 標記 not-ready

- 測試: `TestWaitForTermination_ServerErrorMarksNotReady`
- 假設: server error channel 回傳 error
- 當: lifecycle wait helper 收到該 error
- 那麼: helper返回相同 error，readiness 變成 false

### 場景 D：manifest budget 與 native preStop

- 測試: Kustomize render 驗證
- 假設: base deployment 定義 graceful termination contract
- 當: render base 與 prod overlay
- 那麼: Deployment 都包含 `terminationGracePeriodSeconds: 45` 與 native preStop sleep 5 秒，不含 exec sleep

### 場景 E：既有 cleanup lifecycle 不變

- 測試: `TestStartSyncWorker_CancelAndWait`、`TestWaitForSyncWorker_DeadlineExceeded`
- 假設: Worker 正常回應 cancellation 或忽略 cancellation
- 當: main 啟動及等待 Worker
- 那麼: cancellation/done 與 bounded wait 行為維持不變

## 驗收條件

- [x] `/readyz` 初始回傳 `200`。
- [x] shutdown signal path 在 task 返回前將 readiness 設為 false。
- [x] server error path 在 task 返回前將 readiness 設為 false並保留原 error。
- [x] not-ready 時 `/readyz` 回傳 `503`，不輸出敏感資料。
- [x] `/healthz` 行為維持固定 `200`。
- [x] readiness state 可安全跨 handler goroutine 與 task goroutine存取。
- [x] Deployment 設定 `terminationGracePeriodSeconds: 45`。
- [x] preStop 使用原生 `sleep.seconds: 5`，不使用 exec/shell。
- [x] application cleanup timeout 維持 30 秒，總時間仍保有 10 秒 buffer。
- [x] base 與 prod overlay render 結果一致。
- [x] Worker cancellation/wait 與 cleanup LIFO 順序回歸通過。
- [x] runtime log/HTTP error 保持英文且不含機密值。

## 影響範圍

- `cmd/vault-agent/main.go`
- `cmd/vault-agent/main_test.go`
- `deployments/kustomize/base/deployment.yaml`
- `README.md`
- `docs/deploy.zh-Hant.md`
- 本 spec 文件

## 驗證需求

- readiness 與 lifecycle helper 使用 deterministic unit tests，不送真實 OS signal，不用 sleep 同步。
- 執行 main package race test，驗證 atomic readiness state。
- `kubectl kustomize deployments/kustomize/base` 與 prod overlay都必須成功。
- render 後人工或 script 檢查 termination 45、native sleep 5 與無 exec sleep。
- 在目標 cluster 執行 prod overlay 的 server-side dry-run，確認 Kubernetes API 接受 native lifecycle sleep。
- 執行 `go test -race -count=1 ./...`、`go vet ./...`、`golangci-lint run ./...`。
- 執行 `gofmt -l`、`git diff --check` 與 Boundary 檢查。

## 已知風險

- 目標 cluster 的 server-side dry-run 已接受 native lifecycle sleep；實際 rollout 的 preStop、SIGTERM 與 cleanup 時間軸仍待端對端量測。
- 5 秒是 bounded propagation window，不保證所有網路元件都完成更新。
- readiness 只有在 SIGTERM 送達後變更；preStop 階段的 Endpoint drain 主要依賴 Pod terminating 狀態。
- application cleanup 超過 30 秒時仍會由既有 context timeout 結束，45 秒 Pod budget 不延長 application cleanup。

## 待確認項目

- 無。採 5 秒 native preStop、30 秒 application cleanup 與 45 秒 Pod budget。
