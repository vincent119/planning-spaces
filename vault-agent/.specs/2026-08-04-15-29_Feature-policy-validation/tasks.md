# 任務文件：Policy Validation CLI

Status: Complete

## Execution Context

- 意圖: 新增可在部署前驗證單檔或目錄 policy 的 CLI，並納入 PR CI gate。
- 非目標: 不修改 policy schema、授權決策、hot reload、Vault/AWS live permission 或 Kubernetes deployment。
- 已定決策: 單一 binary；`policy validate`；file/dir 互斥；exit code 0/1/2；重用 `domain.LoadPolicy`；安全 basename 與 rule index；標準庫 `flag`。
- 邊界: main dispatcher、policy domain error location、CLI tests、Makefile、Docker publish workflow、繁中使用文件與本 spec。
- 關鍵檔案: `cmd/vault-agent`、`internal/syncer/domain/policy.go`、`Makefile`、`.github/workflows/docker-publish.yml`
- 完成條件: CLI 驗收情境、domain error安全、Make target、CI gate、文件、race test、vet、lint可執行性報告與 diff 檢查完成。

### Protected Behavior

- 無參數 server startup path 與 graceful lifecycle 不變。
- policy file/dir schema、stable merge、互斥來源與 fail-closed behavior 不變。
- error 不輸出 path prefixes、templates、keys 或完整 YAML。
- Dockerfile、Kubernetes manifest、RBAC 與 secrets 不變。

### 邊界

#### Allowed Changes

- `cmd/vault-agent/main.go`
- `cmd/vault-agent/policy_command.go`
- `cmd/vault-agent/policy_command_test.go`
- `internal/syncer/domain/policy.go`
- `internal/syncer/domain/policy_test.go`
- `Makefile`
- `.github/workflows/docker-publish.yml`
- `docs/config.zh-Hant.md`
- `docs/deploy.zh-Hant.md`
- `.specs/2026-08-04-15-29_Feature-policy-validation/`

#### Forbidden

- 新增第三方 CLI 或 validation dependency
- 修改 policy YAML public schema 或 authorization decisions
- 連線 Vault、AWS 或 Kubernetes 進行 validation
- 輸出 policy resource values、完整 YAML 或 credential
- 修改 Dockerfile、deployment、RBAC 或 `_workspace/`

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 建立 CLI red tests | 無 | Complete | `runCLI` undefined 的 compile failure 已確認 |
| T2 實作 CLI dispatcher 與 validator | T1 | Complete | 重用 loader |
| T3 補強安全錯誤定位 | T1 | Complete | basename 與 rule index |
| T4 加入 Makefile、CI 與文件 | T2、T3 | Complete | 同一 smoke target |
| V1 驗收與回歸驗證 | T2、T3、T4 | Complete | 全套品質 gate |

## 實作任務

- [x] T1 建立 CLI red tests
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/policy_command_test.go`、必要的 `internal/syncer/domain/policy_test.go`
    - Forbidden: production code、workflow 與文件
  - Depends: 無
  - Context: 固定 file/dir success、usage error、validation failure、exit codes、streams 與敏感 marker 不洩漏。
  - Verify:
    - `go test ./cmd/vault-agent -run 'TestRunCLI'` 應先因 `runCLI` 不存在而失敗
    - `git diff --check`

- [x] T2 實作 CLI dispatcher 與 validator
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/main.go`、`cmd/vault-agent/policy_command.go`
    - Forbidden: 修改 server initialization、外部 clients 或 domain validation語意
  - Depends: T1
  - Context: arguments 非空時走 CLI；無參數呼叫原 server body；標準庫 flag；成功0、policy錯誤1、usage錯誤2。
  - Verify:
    - `go test ./cmd/vault-agent -run 'TestRunCLI'`
    - `go test ./cmd/vault-agent -run 'Test(Readiness|Wait|StartSync|BuildWebhook)'`

- [x] T3 補強安全錯誤定位
  - Status: Complete
  - Boundary:
    - Allowed Changes: `internal/syncer/domain/policy.go`、`internal/syncer/domain/policy_test.go`
    - Forbidden: public YAML schema、authorization matching 與 error 內的 resource values
  - Depends: T1
  - Context: decode 時只保存 basename；Validate 產生 group/index；duplicate 顯示目前與首次位置。
  - Verify:
    - `go test ./internal/syncer/domain -run 'TestPolicy'`
    - marker 不出現在 domain 或 CLI error output

- [x] T4 加入 Makefile、CI 與文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: `Makefile`、`.github/workflows/docker-publish.yml`、`docs/config.zh-Hant.md`、`docs/deploy.zh-Hant.md`
    - Forbidden: 新 workflow、修改 image publish triggers、英文文件或 deployment manifest
  - Depends: T2、T3
  - Context: Make target 驗證 canonical file 與 policies dir；workflow 在 Docker build 前執行 target；文件記錄 CLI 與 exit codes。
  - Verify:
    - `make policy-validate`
    - `rg -n 'policy-validate|policy validate' Makefile .github/workflows/docker-publish.yml docs/config.zh-Hant.md docs/deploy.zh-Hant.md`

## 驗證任務

- [x] V1 驗收情境覆蓋
  - Verify: requirements 的 file、dir、invalid、usage 與 server regression 情境均有 automated test 或 smoke command。

- [x] V1 回歸驗證
  - Verify: `go test -race -count=1 ./...`、`go vet ./...`、base/prod Kustomize render。

- [x] V1 品質檢查清單
  - [x] CLI file 與 dir success 通過
  - [x] exit code 0/1/2 contract 通過
  - [x] error location 含 basename 與 rule group/index
  - [x] sensitive path marker 未出現在 output
  - [x] 無參數 server helper tests 通過
  - [x] `make policy-validate` 通過
  - [x] CI 在 image build 前執行 validation
  - [x] Go race test 與 vet 通過
  - [x] lint 執行結果已記錄
  - [x] base/prod Kustomize render 通過
  - [x] 文件一致性已確認
  - [x] `git diff --stat` 已檢查
  - [x] `git diff --check` 已通過

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的 `Execution Context`
2. 目前未完成 task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個 `.specs` 目錄。若文件很大，先用標題與關鍵字定位：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-15-29_Feature-policy-validation
```

## Implementation Notes

- 2026-08-04: 已完成 discovery 與正式 spec，尚未修改 production code、tests、Makefile、workflow 或 user-facing docs。
- 2026-08-04: T1 新增 file、dir、invalid policy 與 usage contract tests；目標測試因 `runCLI` undefined 按預期 compile fail。
- 2026-08-04: T2 已新增標準庫 CLI dispatcher；有 arguments 時不載入 application config，file/dir validation 與 exit code 0/1/2 tests 通過。
- 2026-08-04: T3 已加入 basename、rule group/index 與 duplicate 首次位置；domain 與 CLI marker tests 確認錯誤不輸出 resource values。
- 2026-08-04: 目錄 mode 的 rule index 是穩定排序並合併後的全域 index；duplicate 測試固定 `01.yaml webhook_rules[0]` 與 `02.yaml webhook_rules[1]`。
- 2026-08-04: T4 已新增 `make policy-validate`，單檔與目錄 smoke validation 均通過；Docker publish workflow 在 image build 前執行相同 target，繁中設定與部署文件已更新。
- 2026-08-04: `make build` 原本只編譯 `main.go`，會漏掉同 package 的 CLI 檔案；已在 T4 的 Makefile 邊界內改為編譯 `./cmd/vault-agent/`。
- 2026-08-04: V1 `go test -race -count=1 ./...`、`go vet ./...`、`make build`、實際 binary file/dir smoke、base/prod Kustomize render、gofmt、workflow YAML 與 diff checks 全部通過。
- 2026-08-04: `golangci-lint run ./...` 仍在 package loading 階段回報 `no go files to analyze`；`go list`、build、test 與 vet 正常，依既有問題另案追蹤。
- 2026-08-04: 本機沒有 `actionlint`，改以 Ruby YAML parser 驗證 workflow syntax，並人工確認 validation step 位於 image build 前。
- 既有 `domain.LoadPolicy` 已具備 strict YAML、stable directory sorting、版本一致與 duplicate rule ID 驗證，CLI 必須直接重用。
- `golangci-lint 2.12.1` 在目前環境有既有 `no go files to analyze` loading 問題；V1 仍需執行並如實記錄。
- `_workspace/` 不屬於本 spec，不納入提交。

## 驗證結果摘要

- 新行為驗證: 通過；CLI file/dir、invalid、usage、exit codes、location 與 marker tests 均成功
- 回歸驗證: 通過；全套 race test、vet、binary build 與 base/prod Kustomize render 成功
- 文件一致性: requirements、design、tasks、Makefile、workflow 與繁中使用文件已對齊
- 剩餘風險: golangci-lint package loading 問題仍待另案修復；GitHub-hosted runner 的實際 workflow 執行需由 PR CI 確認

## 後續改善

- [ ] 另案評估 policy hot reload 與 generation 一致性。
- [ ] 另案評估 machine-readable validation output。
- [ ] 另案處理 golangci-lint 專案設定與 package loading 問題。
