# 設計文件：Policy 目錄部署模式

## 設計摘要

本設計在 Alpine prep stage 建立 `/app/policy` 並設定 UID/GID `10000`，再將目錄複製至 Chainguard static final stage。Kubernetes 將 policy ConfigMap 唯讀掛載到該目錄，並透過 `AUTHORIZATION_POLICY_DIR=/app/policy` 啟用既有多檔 loader。實際 policy 保持外部化，不寫入 image，也不改動授權語意。

## 文件定位

本設計實現同目錄 `requirements.md`，接續安全機密存取功能既有的 `policy_dir` 程式能力，只補齊 Docker 與 Kubernetes deployment 契約。不重寫 `internal/configs`、`internal/syncer/domain`、Webhook authentication、Admission contract validation 或 workload authorization。

## 已知契約狀態

- 需求來源: 使用者指定新增 `/app` 與 `/app/policy`
- API / CLI / Hook contract: `AUTHORIZATION_POLICY_FILE` 與 `AUTHORIZATION_POLICY_DIR` 對應既有 Viper 設定，兩者互斥
- Data contract: policy directory 只讀取 `.yaml` 與 `.yml`，各檔案沿用既有 policy schema
- 既有實作: `PolicyDir`、環境變數綁定、多檔載入與規則合併已存在
- 不可假造: 不假設 Docker image 已成功建置；目前只有 Dockerfile 靜態證據與 Kustomize render 證據

## Bounded Context

包含：

- final image 的 `/app` 與 `/app/policy` 目錄
- `/app/policy` 的 non-root ownership
- Kubernetes ConfigMap volume mount 路徑
- deployment 的 authorization policy 環境變數
- 設定範例與繁體中文操作文件

不包含：

- policy parser、排序、合併、衝突或 hot reload
- Vault/AWS server-side policy
- namespace RoleBinding 自動產生
- TLS、Webhook credential 與 Secret mount 遷移
- 實際 workload policy 內容設計

## 設計原則

- policy 與 image 分離，遵循外部化設定。
- final image 維持 non-root 執行，目錄 owner 對齊 UID/GID `10000`。
- Kubernetes 使用唯讀 ConfigMap mount。
- deployment 只選一種 policy source，避免啟動時互斥驗證失敗。
- 保留應用程式層的 `policy_file` 相容能力，不移除既有公開設定。

## 需求對應

| 需求 / 驗收情境 | 設計處理方式 | 驗證方式 |
|-----------------|--------------|----------|
| Final image 提供 policy 目錄 | prep stage 建目錄並以 `COPY --chown` 複製至 final stage | Dockerfile 靜態檢查；發布前 image runtime 檢查 |
| Deployment 使用多檔模式 | mountPath 與 `AUTHORIZATION_POLICY_DIR` 統一為 `/app/policy` | `kubectl kustomize deployments/kustomize/overlays/prod` |
| 設定來源互斥 | 移除 Deployment 的 `AUTHORIZATION_POLICY_FILE` | render 後搜尋兩個環境變數 |
| 既有授權行為不變 | 不修改 Go loader 與 policy domain | `go test -race -count=1 ./...` |

## 受影響檔案計畫

| 檔案 | 預期變更 | 原因 | 風險 |
|------|----------|------|------|
| `Dockerfile` | 建立、複製 `/app/policy` 並設定 `WORKDIR /app` | final image 需要明確掛載點 | 空目錄複製需 image build 驗證 |
| `deployments/kustomize/base/deployment.yaml` | 改 mountPath 與 policy env | 啟用多檔 policy 模式 | 路徑不一致會導致啟動失敗 |
| `configs/config.sample.yaml` | 預設範例改用 `policy_dir` | 對齊 deployment | 本機直接套用範例時需提供目錄 |
| `docs/config.zh-Hant.md` | 說明目錄、ConfigMap key 與副檔名 | 降低配置歧義 | 無 |
| `docs/deploy.zh-Hant.md` | 更新安全部署順序 | 避免同時設定兩種來源 | 無 |

## 目標結構或流程

```text
Docker prep stage
  /app/
    policy/          owner 10000:10000

Docker final stage
  /app/
    policy/          空掛載點
  /vault-agent

Kubernetes Pod
  ConfigMap vault-agent-policy
    -> readOnly mount /app/policy
    -> AUTHORIZATION_POLICY_DIR=/app/policy
    -> 既有 policy directory loader
```

## Mermaid Diagrams

不需要；目錄與資料流使用上方文字結構即可清楚表達。

## 介面與資料契約

### API / CLI / Hook

- Input: `AUTHORIZATION_POLICY_DIR=/app/policy` 與掛載於該目錄的 YAML 檔案
- Output: 啟動時載入並合併既有 workload authorization rules
- Error: 目錄不存在、無可用 YAML、檔案無效或同時設定 `policy_file` 時，沿用既有英文 runtime error

### Data / Config

- 新增資料: 無；只調整既有 policy 檔案位置與來源選擇
- 既有資料相容性: policy schema 不變；使用單檔模式的外部部署仍可設定 `AUTHORIZATION_POLICY_FILE`

## 關鍵行為

- final image 不包含實際 policy，避免環境規則與 image release 綁定。
- ConfigMap volume 覆蓋 `/app/policy` 後，程式讀取 Kubernetes 投影的 YAML 檔案。
- base 修改會由 prod overlay 自然繼承，不新增重複 patch。
- `ENTRYPOINT ["/vault-agent"]` 使用絕對路徑，不受 `WORKDIR /app` 影響。

## 前後端或跨模組設計

Docker image 提供目錄，Kubernetes deployment 提供檔案與環境變數，既有 Go configuration 與 policy loader 消費該目錄。三層路徑固定為 `/app/policy`，避免 mount 與 env 漂移。

## Protected Behavior

- container 持續以 `vault-agent:vault-agent` 執行。
- TLS 與 bearer token 路徑維持 `/etc/vault-agent` 下的既有位置。
- `authorization.enabled` 預設維持 `true`。
- `policy_file` 與 `policy_dir` 仍互斥且至少設定一個。
- base/prod Kustomize、graceful termination、probe 與 resource 設定不變。

## 替代方案

| 方案 | 優點 | 缺點 | 結論 |
|------|------|------|------|
| 保留 `/etc/vault-agent/policy` 單檔模式 | 不需變更 Dockerfile | 未使用既有目錄能力，大量規則維護較集中 | 不採用 |
| 將 policy COPY 進 image | 啟動時不需 ConfigMap | 環境設定與 image 綁定，增加機密與發布風險 | 不採用 |
| 使用 `/app/policy` ConfigMap 目錄模式 | 支援分檔、設定外部化、路徑清楚 | 需要同步 image、mount、env 與文件 | 採用 |

## 風險與處理方式

| 風險 | 影響 | 處理方式 | 驗證 |
|------|------|----------|------|
| Docker 空目錄未進入 final image | 非 Kubernetes 執行時路徑不存在 | prep stage 建立後跨 stage COPY | 發布前 image build 與 `stat /app/policy` |
| mountPath 與 env 不一致 | application 啟動失敗 | 使用單一固定路徑 `/app/policy` | Kustomize render 搜尋 |
| 同時設定 file 與 dir | fail-fast validation 阻止啟動 | deployment 移除 file env | config tests 與 render 搜尋 |
| policy ConfigMap 無有效 YAML | authorization 初始化失敗 | 保留 `policy.yaml`，文件說明副檔名 | manifest review 與既有 loader tests |
| static image 無 shell | 無法用 exec hook 或 shell 建目錄 | build stage 建立並複製目錄 | Dockerfile review |

## 實作注意事項

- 本 spec 是在 commit `83c3bfa` 完成實作後補建，文件不得宣稱遵循先 spec 後 implementation 的順序。
- Docker daemon 未啟動，不把靜態 Dockerfile 檢查描述成實際 image runtime 驗證。
- `_workspace/` 不屬於本 spec，不納入提交。
