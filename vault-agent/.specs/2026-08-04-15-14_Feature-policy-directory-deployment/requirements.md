# 需求文件：Policy 目錄部署模式

## 來源

- Draft: 無
- Type: Feature
- Owner: Vincent
- Status: Complete

## 文件定位

本 spec 接續安全機密存取功能對 `authorization.policy_dir` 的應用程式支援，補齊容器 image 與 Kubernetes deployment 的目錄契約。範圍只處理 `/app/policy`、ConfigMap 目錄掛載及部署設定，不重寫 policy parser、workload authorization、caller authentication、AdmissionReview validation 或 Kubernetes RBAC。

參考來源：

- 需求來源: 使用者要求新增 `/app` 與 `/app/policy`
- 既有文件: `docs/config.zh-Hant.md`、`docs/deploy.zh-Hant.md`
- 既有程式碼: `Dockerfile`、`deployments/kustomize/base/deployment.yaml`、`internal/configs/config.go`

## 背景

應用程式已支援 `authorization.policy_file` 與 `authorization.policy_dir` 二擇一，但 Kubernetes deployment 只設定 `AUTHORIZATION_POLICY_FILE=/etc/vault-agent/policy/policy.yaml`。大量 namespace 的規則若集中在單一檔案，維護與分工成本較高；容器最終 stage 也沒有明確建立可供 policy 掛載的應用程式目錄。

## 問題陳述

目前 deployment 沒有使用既有的多檔 policy 載入能力，且 final image 未明確提供 `/app/policy` 目錄。部署契約與建議的多檔 policy 管理方式不一致。

## 目標

1. final image 明確包含 `/app` 與 `/app/policy`。
2. `/app/policy` 的 owner 對齊 runtime UID/GID `10000`。
3. Kubernetes deployment 將 policy ConfigMap 唯讀掛載到 `/app/policy`。
4. deployment 使用 `AUTHORIZATION_POLICY_DIR=/app/policy`，不再同時設定 `AUTHORIZATION_POLICY_FILE`。
5. 設定範例與繁體中文文件對齊新的預設部署方式。

## 非目標

1. 不新增 policy hot reload。
2. 不新增 policy schema validation CLI。
3. 不修改 policy 合併、驗證或授權語意。
4. 不把實際 workload policy 烘焙進 container image。
5. 不移動 TLS 或 Webhook bearer token 的既有掛載路徑。

## 已定決策

- 目錄固定為 `/app/policy`。
- policy 由 Kubernetes ConfigMap 唯讀掛載，image 只建立空目錄。
- Kubernetes 預設使用 `policy_dir`；應用程式仍保留 `policy_file` 相容能力。
- `policy_file` 與 `policy_dir` 持續採二擇一驗證。

## 待確認項目

- 無。

## 現有行為

- builder stage 使用 `/app`，final static image 未複製或建立 `/app`。
- policy ConfigMap 掛載至 `/etc/vault-agent/policy`。
- deployment 設定 `AUTHORIZATION_POLICY_FILE=/etc/vault-agent/policy/policy.yaml`。
- ConfigMap 只有一個 `policy.yaml` 時可運作，但 deployment 未啟用目錄載入模式。

## 新行為

- prep stage 建立 `/app/policy`，設定 owner 為 `vault-agent:vault-agent`，再複製到 final image。
- final image 的工作目錄為 `/app`。
- policy ConfigMap 唯讀掛載至 `/app/policy`。
- deployment 只設定 `AUTHORIZATION_POLICY_DIR=/app/policy`。
- ConfigMap 可透過多個 `.yaml` 或 `.yml` key 提供分檔 policy。

## 影響範圍

- 使用者: 維護 Kubernetes workload authorization policy 的平台管理者
- 功能: container image 目錄、Kubernetes policy ConfigMap 掛載與 authorization 設定來源
- API / CLI: 無
- Data / Storage: policy YAML 內容不變，只調整檔案掛載目錄
- 文件 / 安裝 / 發布: Docker image build、Kustomize deployment、設定與部署文件

## 使用情境

- 作為平台管理者，我想把不同團隊或 namespace 的 policy 拆成多個 YAML 並掛載至 `/app/policy`，以便降低單一大型 policy 檔案的維護成本。

## 驗收情境

### 情境：Final image 提供 policy 目錄

- 場景：以 non-root 使用者啟動 vault-agent image
- 測試：Dockerfile 靜態檢查；image runtime 檢查列入發布前驗證
- 假設：image 依專案 Dockerfile 建置
- 當：檢查 final stage 的檔案與工作目錄契約
- 那麼：存在 `/app/policy`，其 owner 為 UID/GID `10000`，工作目錄為 `/app`

### 情境：Deployment 使用多檔 policy 模式

- 場景：render base 或 prod Kustomize manifest
- 測試：`kubectl kustomize deployments/kustomize/overlays/prod`
- 假設：base Deployment 掛載 `vault-agent-policy` ConfigMap
- 當：render Deployment
- 那麼：只出現 `AUTHORIZATION_POLICY_DIR=/app/policy` 與唯讀 `/app/policy` mount，不出現 `AUTHORIZATION_POLICY_FILE`

### 情境：既有授權設定驗證不被破壞

- 場景：驗證 `policy_file` 與 `policy_dir` 的互斥契約
- 測試：`go test ./internal/configs ./internal/syncer/domain`
- 假設：應用程式既有設定與 policy loader 未修改
- 當：執行既有測試
- 那麼：單檔、目錄與互斥驗證維持通過

## 驗收條件

1. Dockerfile 建立並複製 `/app/policy` 至 final stage，owner 為 UID/GID `10000`。
2. final stage 設定 `WORKDIR /app`，既有 `ENTRYPOINT ["/vault-agent"]` 不變。
3. base 與 prod render 只包含 `AUTHORIZATION_POLICY_DIR=/app/policy`。
4. policy volume mount 使用 `/app/policy` 且為唯讀。
5. Go race test、vet、Kustomize render 與 `git diff --check` 通過。
6. 設定與部署文件說明 Kubernetes 預設目錄模式及二擇一限制。

## 驗證需求

- Unit / Integration: `go test -race -count=1 ./...`
- CLI / Dry-run: `kubectl kustomize deployments/kustomize/base`、`kubectl kustomize deployments/kustomize/overlays/prod`
- 文件檢查: 搜尋 `/app/policy`、`AUTHORIZATION_POLICY_DIR` 與殘留的舊 policy mount
- 回歸驗證: `go vet ./...`、`go test ./internal/configs ./internal/syncer/domain`

## 風險與假設

| 類型 | 內容 | 處理方式 |
|------|------|----------|
| 風險 | 空目錄跨 Docker stage 複製行為尚未以本機 daemon 實際建置確認 | 發布前啟動 Docker daemon，建置 image 並檢查 `/app/policy` |
| 風險 | 掛載 ConfigMap 會遮蔽 image 內同一路徑 | image 目錄只作預設掛載點，不放置內建 policy |
| 假設 | 目標 Kubernetes 支援 ConfigMap directory mount | 既有 deployment 已使用相同掛載機制，並以 Kustomize render 驗證 manifest |

## 摘要

- 關鍵決策: final image 建立 `/app/policy`，Kubernetes 預設改用 `AUTHORIZATION_POLICY_DIR`
- 待確認項目: 無
- 風險: Docker daemon 未啟動，尚未執行實際 image build 與 runtime 目錄檢查
- 下一步: 發布前補做 image build 驗證；其餘本 spec 實作與文件已完成
