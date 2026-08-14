# 任務文件：Policy 目錄部署模式

Status: Complete

## Execution Context

- 意圖: 在 final image 建立 `/app/policy`，並讓 Kubernetes 預設使用該目錄載入多檔 workload policy。
- 非目標: 不修改 Go policy loader、授權語意、RBAC、hot reload 或實際 policy 規則。
- 已定決策: `/app/policy`；UID/GID `10000`；ConfigMap 唯讀掛載；只設定 `AUTHORIZATION_POLICY_DIR`；policy 不寫入 image。
- 邊界: 只修改 Dockerfile、base Deployment、設定範例、繁中設定與部署文件，以及本 spec。
- 關鍵檔案: `Dockerfile`、`deployments/kustomize/base/deployment.yaml`、`configs/config.sample.yaml`、`docs/config.zh-Hant.md`、`docs/deploy.zh-Hant.md`
- 完成條件: Dockerfile、base/prod render、Go 回歸測試、vet、文件與 diff 檢查通過；未完成的實際 image build 必須明確記錄。

### Protected Behavior

- runtime 使用者維持 UID/GID `10000`。
- `ENTRYPOINT ["/vault-agent"]`、TLS、bearer token、probe、resource 與 graceful termination 設定不變。
- `authorization.enabled=true` 與 policy source 互斥驗證不變。
- application 的 policy schema、目錄 loader 與授權行為不變。

### 邊界

#### Allowed Changes

- `Dockerfile`
- `deployments/kustomize/base/deployment.yaml`
- `configs/config.sample.yaml`
- `docs/config.zh-Hant.md`
- `docs/deploy.zh-Hant.md`
- `.specs/2026-08-04-15-14_Feature-policy-directory-deployment/`

#### Forbidden

- 修改 `internal/configs` 或 `internal/syncer` production code
- 修改 policy schema 或 merge order
- 把實際 policy、credential 或 secret 寫入 image
- 修改 TLS、Webhook authentication、RBAC 或 graceful termination
- 納入 `_workspace/`

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 建立 final image policy 目錄 | 無 | Complete | commit `83c3bfa` |
| T2 切換 Kubernetes policy directory mode | T1 | Complete | base 與 prod render 通過 |
| T3 更新設定與部署文件 | T2 | Complete | 繁中文件已更新 |
| V1 執行整合與回歸驗證 | T1、T2、T3 | Complete | Docker runtime 驗證列為發布前風險 |
| V2 補建 SDD | V1 | Complete | 本 spec 誠實記錄後補順序 |

## 實作任務

- [x] T1 建立 final image policy 目錄
  - Status: Complete
  - Boundary:
    - Allowed Changes: `Dockerfile`
    - Forbidden: 改變 binary 路徑、runtime 使用者或把 policy 寫入 image
  - Depends: 無
  - Context: Chainguard static final image 無 shell，需在 Alpine prep stage 建立目錄後複製。
  - Verify:
    - `rg -n 'mkdir -p /app/policy|COPY --from=prep --chown=10000:10000 /app /app|WORKDIR /app' Dockerfile`
    - 發布前建置 image 並檢查 `/app/policy` owner；本機 Docker daemon 未啟動，尚未執行

- [x] T2 切換 Kubernetes policy directory mode
  - Status: Complete
  - Boundary:
    - Allowed Changes: `deployments/kustomize/base/deployment.yaml`、必要的既有 policy ConfigMap
    - Forbidden: 修改 TLS、bearer token、RBAC、probe、resource 或 graceful termination
  - Depends: T1
  - Context: ConfigMap 投影目錄需與 `AUTHORIZATION_POLICY_DIR` 完全一致，且不得保留 `AUTHORIZATION_POLICY_FILE`。
  - Verify:
    - `kubectl kustomize deployments/kustomize/base`
    - `kubectl kustomize deployments/kustomize/overlays/prod`
    - render 結果只包含 `AUTHORIZATION_POLICY_DIR=/app/policy` 與唯讀 `/app/policy` mount

- [x] T3 更新設定與部署文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: `configs/config.sample.yaml`、`docs/config.zh-Hant.md`、`docs/deploy.zh-Hant.md`
    - Forbidden: 修改英文文件或宣稱已完成 Docker runtime 驗證
  - Depends: T2
  - Context: 文件需區分 Kubernetes 預設目錄模式與應用程式保留的單檔能力。
  - Verify:
    - `rg -n '/app/policy|AUTHORIZATION_POLICY_DIR|policy_file|policy_dir' configs/config.sample.yaml docs/config.zh-Hant.md docs/deploy.zh-Hant.md`

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Verify: Dockerfile 靜態契約、base/prod render 與既有 config/domain tests 已覆蓋 requirements 的三個主要情境；image runtime 檢查明確列為發布前驗證。

- [x] V1 回歸驗證
  - Verify: `go test -race -count=1 ./...`、`go vet ./...` 與 `go test ./internal/configs ./internal/syncer/domain` 通過。

- [x] V1 品質檢查清單
  - [x] Dockerfile 明確建立並複製 `/app/policy`
  - [x] runtime UID/GID 維持 `10000`
  - [x] base/prod Kustomize render 通過
  - [x] render 只設定 `AUTHORIZATION_POLICY_DIR=/app/policy`
  - [x] policy mount 為唯讀 `/app/policy`
  - [x] 既有 Go race test與 vet 通過
  - [x] 文件一致性已確認
  - [x] Protected Behavior 回歸檢查通過
  - [x] `git diff --stat` 已檢查
  - [x] `git diff --check` 已通過

- [x] V2 補建 SDD
  - Status: Complete
  - Boundary:
    - Allowed Changes: 本 spec 三份文件
    - Forbidden: 回寫或重製 commit `83c3bfa` 的實作
  - Depends: V1
  - Context: 使用者指出先前提交未納入正式 SDD，需依專案格式補齊並記錄實際順序。
  - Verify:
    - 三份文件包含 requirements、design、tasks 必要章節
    - `git diff --check`

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的 `Execution Context`
2. 目前未完成 task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個 `.specs` 目錄。若文件很大，先用標題與關鍵字定位：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-15-14_Feature-policy-directory-deployment
```

## Implementation Notes

- 2026-08-04: 實作先於 spec 完成，commit 為 `83c3bfa feat: 新增 policy 目錄部署模式`；本文件為使用者要求後補，不宣稱原流程符合 spec-first。
- 2026-08-04: Dockerfile 在 prep stage 建立 `/app/policy` 並設定 owner，再以 `COPY --chown=10000:10000` 複製至 final stage，`WORKDIR` 設為 `/app`。
- 2026-08-04: base Deployment 已改用唯讀 `/app/policy` mount 與 `AUTHORIZATION_POLICY_DIR=/app/policy`；prod overlay 直接繼承且 render 成功。
- 2026-08-04: `go test -race -count=1 ./...`、`go vet ./...`、config/domain tests、base/prod Kustomize 與 `git diff --check` 通過。
- 2026-08-04: Docker client `29.1.3` 存在，但 Colima daemon socket 不存在，無法執行實際 image build。
- 2026-08-04: `golangci-lint 2.12.1` 使用 Go `1.26.2` 建置，在專案 Go `1.26.5` 環境回報 `no go files to analyze`；`go list ./...` 正常。此問題未歸因於本次 Docker/YAML 變更。
- `_workspace/` 不屬於本 spec，不納入提交。

## 驗證結果摘要

- 新行為驗證: Dockerfile 靜態契約與 base/prod Kustomize render 通過
- 回歸驗證: 全套 race test、vet 與 config/domain tests 通過
- 文件一致性: `configs/config.sample.yaml`、繁中設定文件、部署文件與本 spec 已對齊
- 剩餘風險: 實際 image build/runtime 目錄檢查未執行；golangci-lint package loading 問題待另案處理

## 後續改善

- [ ] Docker daemon 可用後，建置 image 並以 runtime 檢查確認 `/app/policy` 與 UID/GID。
- [ ] 另案建立可重現的 golangci-lint 專案設定並修正 package loading 問題。
- [ ] 另案評估 policy schema validation CLI 與 policy hot reload。
