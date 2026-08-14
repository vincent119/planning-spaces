# 任務文件：golangci-lint 可重現設定與 Package Loading 診斷

Status: Complete

## Execution Context

- 意圖: 建立repository-owned golangci-lint v2 config與單一版本來源，修正真實errcheck findings，讓local/CI lint可重現。
- 非目標: 不升級Go/dependencies、不加入主要go.mod tool、不擴張linters、不修改業務或deployment behavior。
- 已定決策: v2.12.2官方binary；`.golangci-lint-version`單一來源；explicit 5 linters；readonly modules；Makefile version guard；獨立read-only CI workflow；不以exclusion掩蓋4個errcheck。
- 邊界: lint config/version、Makefile、policy CLI writer handling/tests、獨立workflow、開發文件與本spec。
- 關鍵檔案: `.golangci.yml`、`.golangci-lint-version`、`Makefile`、`policy_command.go/test`、`.github/workflows/golangci-lint.yml`。
- 完成條件: requirements A至G、本機固定版本`make lint`、race/vet/policy tests、config/YAML verify、Boundary與diff checks通過；CI runtime結果只在實際run後記錄。

### Protected Behavior

- policy CLI success=0、validation failure=1、usage failure=2維持；只有success stdout write failure新增exit1。
- CLI usage、success/error訊息維持英文，invalid policy不得洩露敏感值。
- policy file/directory validation與schema不變。
- Go 1.26.5、go.mod/go.sum與production dependencies不變。
- Webhook、authentication、authorization、Secret sync、backend與Kubernetes manifests不變。
- Docker publish workflow既有build/push/tag behavior不變。
- runtime與CI不得下載`latest`或依賴home config。

### 邊界

#### Allowed Changes

- `.golangci.yml`
- `.golangci-lint-version`
- `Makefile`
- `cmd/vault-agent/policy_command.go`
- `cmd/vault-agent/policy_command_test.go`
- `.github/workflows/golangci-lint.yml`
- `README.md`
- `docs/README.zh-Hant.md`
- `docs/README.en.md`
- `.specs/2026-08-04-16-55_Chore-golangci-lint-reproducibility/`

#### Forbidden

- 修改go.mod、go.sum、Go版本或任何dependency
- 新增tools.go、go tool directive或從source buildgolangci-lint
- 修改Dockerfile、docker-publish workflow語意或deployments
- 新增`linters.default: all`、廣域exclusion、baseline、`nolint`以掩蓋現有finding
- 修改policy schema、path/authz/security behavior或機密錯誤遮蔽
- 硬編碼home、sandbox、runner或個人cache絕對路徑
- 自動安裝`latest`、執行release/tag/push
- 納入`_workspace/`

## 任務依賴

| 任務 | Depends | 狀態 | 備註 |
|------|---------|------|------|
| T1 固定package loading與lint baseline | 無 | Complete | go list、verbose lint、4 errcheck已固定 |
| T2 建立v2 config與版本來源 | T1 | Complete | schema、5 linters、v2.12.2 |
| T3 修正policy CLI writer findings | T1、T2 | Complete | writer behavior tests與CLI lint通過 |
| T4 固定Makefile lint入口 | T2、T3 | Complete | mismatch/missing binary皆快速失敗 |
| T5 建立CI lint gate與開發文件 | T4 | Complete | separate read-only workflow |
| V1 config、version與loader驗證 | T2、T4、T5 | Complete | v2.12.2 verify、9 packages、5 linters、0 issues |
| V2 behavior與全專案回歸 | T3至T5 | Complete | race/vet/policy/YAML/Boundary與實際CI runtime |

## 實作任務

- [x] T1 固定package loading與lint baseline
  - Status: Complete
  - Boundary:
    - Allowed Changes: 本spec Implementation Notes
    - Forbidden: 修改config、Makefile、Go files、module files或cache設定
  - Depends: 無
  - Context: 從repository root記錄Go/tool版本、go env、`go list ./...`、`golangci-lint run -v ./...`與exit code。區分受限環境loader症狀和具cache權限環境的真實4個errcheck；不得執行`go mod tidy`。
  - Verify:
    - `go version`
    - `golangci-lint version`
    - `go env GOMOD GOWORK GOFLAGS GOCACHE GOTOOLCHAIN`
    - `go list ./...`
    - `golangci-lint run -v ./...`
    - baseline必須記錄9個packages、5個active linters與4個`policy_command.go` errcheck。
    - `git diff --stat`、`git diff --check`。

- [x] T2 建立v2 config與版本來源
  - Status: Complete
  - Boundary:
    - Allowed Changes: `.golangci.yml`、`.golangci-lint-version`、本spec Implementation Notes
    - Forbidden: Makefile、Go files、workflow、module files、exclusions或額外linters
  - Depends: T1
  - Context: 新增v2 schema、readonly modules、default none、明確5 linters、unlimited issue output；版本檔固定`v2.12.2`。版本檔最後一行需有newline。
  - Verify:
    - `golangci-lint config verify -c .golangci.yml`
    - `golangci-lint config path -c .golangci.yml`或v2.12.2相等命令確認repository config。
    - structured YAML assertions確認version、default、linters、module mode與無exclusions。
    - `test "$(cat .golangci-lint-version)" = "v2.12.2"`。
    - `git diff --stat`、`git diff --check`。

- [x] T3 修正policy CLI writer findings
  - Status: Complete
  - Boundary:
    - Allowed Changes: `cmd/vault-agent/policy_command.go`、`policy_command_test.go`、本spec Implementation Notes
    - Forbidden: policy domain、schema、loading、其他CLI/server files、nolint或config exclusion
  - Depends: T1、T2
  - Context: 先新增failing writer red tests。Failure path output採局部best effort並保留exit1/2；success stdout write失敗時best-effort寫英文stderr並返回1。不得輸出policy值或secret。
  - Verify:
    - `go test ./cmd/vault-agent -run 'TestRunCLI_(ValidatePolicySuccessWriterFailure|ErrorWriterFailurePreservesExitCode|ValidatePolicyFileSuccess|ValidatePolicyDirectorySuccess|InvalidPolicyReportsLocationWithoutSensitiveValues|UsageErrors)'`
    - production修改前新tests應按預期失敗，red evidence記錄至Implementation Notes。
    - `golangci-lint run -c .golangci.yml ./cmd/vault-agent/...`不再回報4個finding。
    - `gofmt -l cmd/vault-agent/policy_command.go cmd/vault-agent/policy_command_test.go`無輸出。
    - `git diff --stat`、`git diff --check`。

- [x] T4 固定Makefile lint入口
  - Status: Complete
  - Boundary:
    - Allowed Changes: `Makefile`、必要的test-only version guard fixture策略、本spec Implementation Notes
    - Forbidden: 自動下載、latest、bash-only語法、home/cache path、go.mod/go.sum
  - Depends: T2、T3
  - Context: `GOLANGCI_LINT`可覆寫binary path；expected version只讀版本檔。`lint`依賴version guard並明確`--config .golangci.yml`。缺少binary或版本不符時用繁體中文快速失敗，不執行analysis。
  - Verify:
    - 正確v2.12.2：`make lint`返回0。
    - fake/mismatched binary：version guard非0，輸出預期與實際版本，且fake run未被呼叫。
    - `make test`、`make vet`與其他targets定義不變。
    - `git diff --stat`、`git diff --check`。

- [x] T5 建立CI lint gate與開發文件
  - Status: Complete
  - Boundary:
    - Allowed Changes: `.github/workflows/golangci-lint.yml`、README與雙語README、spec Implementation Notes
    - Forbidden: 修改docker-publish workflow、packages write、secrets、only-new-issues或重複硬編碼工具版本
  - Depends: T4
  - Context: 獨立workflow使用contents read、go.mod與`.golangci-lint-version`、官方action v9；config verify保持啟用。文件說明官方binary、版本檔、make lint、config verify及可攜cache排障，不放個人路徑。
  - Verify:
    - YAML parse成功。
    - structured assertions確認PR/main triggers、contents read、setup-go讀go.mod、action v9、version-file與無packages write。
    - `rg`確認版本字串只在版本檔與spec出現，不在Makefile/workflow/docs重複。
    - 繁中與既有英文開發段落語意一致。
    - `git diff --stat`、`git diff --check`。

## 驗證任務

- [x] V1 config、version與loader驗證
  - Status: Complete
  - Boundary:
    - Allowed Changes: task邊界內config、Makefile、workflow、docs與Implementation Notes
    - Forbidden: 修改Go/module files來規避loader，使用錯誤工具版本宣稱通過
  - Depends: T2、T4、T5
  - Context: 使用v2.12.2官方binary驗證config、version guard、package loading與完整lint。若下載binary，需要使用者明確授權網路與外部檔案寫入。
  - Verify:
    - `go list ./...`
    - `golangci-lint config verify -c .golangci.yml`
    - `golangci-lint run -v -c .golangci.yml ./...`
    - `make lint`
    - verbose output顯示repository config、9個packages可載入、5個linters且0 issues。

- [x] V2 behavior與全專案回歸
  - Status: Complete
  - Boundary:
    - Allowed Changes: 僅修正本Chore直接造成的邊界內失敗，先記錄Implementation Notes
    - Forbidden: 擴張linters、修改internal/deployments/dependencies或假造CI runtime
  - Depends: T3至T5
  - Context: 驗證CLI writer semantics、全專案behavior、workflow/config syntax與Boundary。
  - Verify:
    - `go test -race -count=1 ./...`
    - `go vet ./...`
    - `make policy-validate`
    - `gofmt -l cmd/vault-agent`無輸出。
    - `.golangci.yml`與workflow YAML parse成功。
    - `git diff --stat`、`git diff --check`。
    - `git status --short`確認`_workspace/`未納入。
    - push後只記錄實際GitHub Actions結果；本機不得代替。

- [x] 品質檢查清單
  - [x] go list成功列出9個packages
  - [x] 受限loader症狀與真實lint findings分開記錄
  - [x] v2 config schema verify通過
  - [x] explicit 5 linters且無廣域exclusion
  - [x] modules download mode為readonly
  - [x] 版本檔固定v2.12.2
  - [x] Makefile與CI共用版本檔
  - [x] 版本不符時make lint快速失敗
  - [x] 4個errcheck已由behavior fix消除
  - [x] success writer failure回傳1
  - [x] failure writer error保留原exit code且不panic
  - [x] CLI訊息為英文且不洩密
  - [x] make lint以固定版本通過
  - [x] CI lint workflow只有contents read
  - [x] 實際GitHub Actions lint workflow通過
  - [x] Docker publish workflow語意不變
  - [x] README與雙語開發文件一致
  - [x] Go/module/deployment無diff
  - [x] race、vet、policy validation通過
  - [x] YAML、格式與diff檢查通過
  - [x] `_workspace/`未納入

## 實作中斷恢復

恢復時優先讀取：

1. 本文件的`Execution Context`
2. 目前未完成task
3. `Protected Behavior`
4. `Implementation Notes`

不得預設掃描整個`.specs`目錄。需要定位時使用：

```bash
rg -n "^#|^##|^###|Boundary:|Depends:|Implementation Notes|Status:" .specs/2026-08-04-16-55_Chore-golangci-lint-reproducibility
```

## Implementation Notes

- 2026-08-04: 規劃前唯讀discovery確認`go list ./...`成功列出9個packages，go.mod為Go 1.26.5。
- 2026-08-04: 本機PATH工具為golangci-lint v2.12.1，built with Go 1.26.2；repository沒有golangci config或版本檔。
- 2026-08-04: 在具cache權限環境執行verbose lint成功完成package loading，啟用5個default linters並回報`policy_command.go`的4個errcheck。
- 2026-08-04: 先前`no go files to analyze`只在受限執行環境出現；目前證據證明repository packages存在，但尚未把golangci內部generic error精確歸因到單一cache syscall。
- 2026-08-04: 官方文件要求v2 config使用`version: "2"`，建議固定官方binary release，且不建議以主要Go module的tool directive安裝。
- 2026-08-04: 官方目前文件列出v2.12.2；設計採`.golangci-lint-version`作為Makefile與CI單一版本來源。
- 2026-08-04: 使用者本次只要求需求規劃task，尚未新增config/version、修改Makefile/Go/workflow/docs或執行外部寫入。
- 2026-08-04: T1重新執行確認Go 1.26.5、golangci-lint v2.12.1、9個packages與5個active linters；loader在633ms完成並只回報`policy_command.go`的4個errcheck。
- 2026-08-04: T2新增v2 config與v2.12.2版本檔；本機v2.12.1的schema verify、config path與結構化contract assertions通過，最終仍待v2.12.2驗收。
- 2026-08-04: T3 red test確認有效policy的stdout writer失敗仍回傳0；其他usage/validation failure writer測試維持原exit2/1且未panic。
- 2026-08-04: T3以局部best-effort helper修正4個errcheck；success stdout failure現在best-effort寫安全英文stderr並回傳1，目標tests與CLI package lint通過。
- 2026-08-04: T4 Makefile已讀取單一版本檔、支援`GOLANGCI_LINT`binary path覆寫並明確指定repository config；本機v2.12.1 mismatch與不存在binary案例皆在analysis前清楚失敗。
- 2026-08-04: T5新增獨立lint workflow，確認PR/main triggers、contents read、go.mod、action v9、version-file與explicit config；Docker publish workflow未變更，雙語開發文件未重複硬編碼版本。
- 2026-08-04: V1下載官方v2.12.2 darwin/arm64 archive到`/private/tmp`並以官方checksums驗證；binary built with Go 1.26.2，沒有寫入repository或系統PATH。
- 2026-08-04: V1以v2.12.2通過config verify/path、9-package`go list`、5-linter verbose run與Makefile version guard；完整lint為0 issues。
- 2026-08-04: V2本機全專案race tests、vet、兩種policy validation、gofmt、config/workflow YAML contracts、no-drift Boundary與`git diff --check`通過。
- 2026-08-05: PR #9實際GitHub Actions已確認通過；`lint`於34秒完成，`build-and-push`於2分50秒完成。V2與整體spec改為Complete。
- 2026-08-05: CI證據：[lint job 91946527142](https://github.com/vincent119/vault-agent/actions/runs/30895260255/job/91946527142)與[build-and-push job 91946526849](https://github.com/vincent119/vault-agent/actions/runs/30895260253/job/91946526849)皆為pass。
- `_workspace/`不屬於本spec，不納入提交。

## 驗證結果摘要

- 新行為驗證: v2.12.2 config/version guard、9-package loader、5 linters、writer failure semantics與完整0-issue lint均通過。
- baseline: 修正前verbose lint找到4個errcheck；修正後固定版本完整lint為0 issues。
- 回歸驗證: 全專案race、vet、policy validation、format、YAML與Boundary檢查通過。
- 剩餘風險: 本spec定義的local與CI驗收均已完成；後續只保留獨立改善項目。

## 後續改善

- [ ] 建立定期golangci-lint patch/minor升級流程，每次以獨立PR檢視新增findings。
- [ ] 評估是否把`make lint`與`make test`整合為單一local verify target。
- [ ] 評估CI action major以commit SHA固定，降低第三方action tag漂移風險。
