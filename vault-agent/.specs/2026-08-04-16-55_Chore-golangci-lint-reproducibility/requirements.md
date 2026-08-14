# 需求文件：golangci-lint 可重現設定與 Package Loading 診斷

## 來源

- Draft: 無
- Type: Chore
- Owner: Vincent
- Status: InProgress

## 文件定位

本 spec 接續多個功能與修復 spec 反覆記錄的 `golangci-lint` 品質閘門問題，包含 `.specs/2026-08-04-15-29_Feature-policy-validation/`、`.specs/2026-08-04-15-14_Feature-policy-directory-deployment/` 與 `.specs/2026-08-04-16-19_BugFix-webhook-startup-readiness/`。本次只建立 repository-owned 的 golangci-lint v2 設定、版本固定、CLI lint findings修正、Makefile與CI驗證契約，不重寫業務邏輯、更新Go版本或調整Kubernetes部署。

參考來源：

- 現有Makefile：`make lint`直接執行`golangci-lint run ./...`
- 現有設定：repository沒有`.golangci.yml`、`.golangci.yaml`或版本檔
- 現有Go module：`go.mod`使用Go `1.26.5`
- 本機工具：`golangci-lint v2.12.1`，以Go `1.26.2`建置
- 官方v2設定：config必須包含`version: "2"`
- 官方安裝建議：固定binary release；不建議把golangci-lint加入主要Go module的`tool` dependencies
- 現有CI：Docker publish workflow沒有lint job或step

## Discovery 證據

2026-08-04在repository root執行唯讀診斷：

1. `go list ./...`成功列出9個packages。
2. 在具備Go build cache權限的環境執行`golangci-lint run -v ./...`，package loader成功完成，啟用`errcheck`、`govet`、`ineffassign`、`staticcheck`與`unused`。
3. verbose lint回報`cmd/vault-agent/policy_command.go`共4個未檢查的`fmt.Fprintln`／`fmt.Fprintf`回傳error，均屬`errcheck`。
4. 受限執行環境曾回報`no go files to analyze`；同一worktree在具備cache權限的環境可成功load，因此目前證據不支持「repository沒有可分析Go package」為根本原因。
5. 受限環境的確切失敗層仍需以`GOCACHE`、`GOLANGCI_LINT_CACHE`與verbose output重現；不得以`go mod tidy`作為未證實的修復。

## 背景

專案目前依賴golangci-lint的v2預設設定。沒有repository config時，工具會向上搜尋父目錄與使用者home設定，且不同版本可能改變預設linters或行為；因此兩位開發者即使執行相同`make lint`，結果也可能受到binary版本與外部config影響。

另一方面，先前在受限執行環境看到的`no go files to analyze`掩蓋了真正的lint findings。verbose evidence已證明Go package可正常載入，並揭露4個`errcheck`。若只新增exclusion或把輸出error全部丟棄，雖可讓lint變綠，卻會讓CLI在stdout寫入失敗時仍回報成功，破壞exit code可信度。

專案現有GitHub Actions只驗證policy並建置image，沒有獨立lint gate。工具版本也未由repository固定，無法保證local與CI使用相同golangci-lint行為。

## 現有行為

- `make lint`依賴PATH中的任意`golangci-lint`版本。
- 工具自動搜尋repository上層或home config，repository沒有明確ownership。
- v2.12.1預設啟用5個linters，但未由專案契約固定。
- `runCLI`未檢查4個格式化輸出的write error。
- CI不執行golangci-lint，PR可在lint失敗時繼續image build。
- 受限環境package loading失敗時沒有標準診斷命令或cache排障指引。

## 新行為

- repository新增`.golangci.yml`，使用v2 schema並明確列出linters。
- repository新增`.golangci-lint-version`，固定官方binary release `v2.12.2`。
- `make lint`只接受版本檔指定的binary版本，並明確使用repository config。
- `.golangci.yml`禁止依賴home或parent config，使用readonly module loading並固定issue輸出契約。
- 修正policy validation CLI的4個`errcheck`，成功輸出失敗時不得回傳exit code 0；best-effort usage/error輸出必須以局部、可說明的方式明確處理。
- 新增CLI writer failure測試，避免只為lint靜默忽略重要error。
- 新增獨立GitHub Actions lint workflow，以版本檔安裝官方binary並驗證v2 config。
- README與既有中英文開發文件記錄安裝版本、`make lint`、config verify與受限cache排障界線。

## 目標

1. 讓local與CI使用相同golangci-lint v2版本與repository config。
2. 讓`go list ./...`與verbose lint成為package loading的標準診斷證據。
3. 明確固定linters，避免工具升級或home config造成結果漂移。
4. 修正目前4個真實`errcheck`，不使用全域exclusion或generated baseline掩蓋。
5. 讓`make lint`在版本正確時可重現通過，版本不符時快速且清楚失敗。
6. 讓CI在PR與main push執行獨立lint gate，不繼承image publish的packages write權限。
7. 保留所有既有CLI正常輸出、usage、exit code與機密遮蔽契約。

## 非目標

1. 不升級或降級Go、production dependencies或golangci-lint內部linters。
2. 不把golangci-lint加入主要`go.mod`的`tool` directive或tools dependency。
3. 不在Makefile硬編碼個人home、Go cache或Codex sandbox路徑。
4. 不新增大批風格linters、全專案格式重寫或歷史debt baseline。
5. 不使用`linters.default: all`，避免新版本加入linter時無預警阻擋。
6. 不使用全域`errcheck` exclusion略過`fmt`、`io.Writer`或CLI package。
7. 不修改policy validation的policy schema、path validation、security訊息或secret處理。
8. 不改Docker image、Kubernetes manifests、Webhook、Sync Worker或backend clients。
9. 不執行release、tag、push或外部系統寫入。

## 使用情境

### 情境一：正確版本執行local lint

開發者安裝版本檔指定的官方binary並執行`make lint`。Makefile驗證版本，golangci-lint讀取repository config，所有packages載入且沒有lint findings，命令返回0。

### 情境二：工具版本不符

PATH中的golangci-lint不是`v2.12.2`。`make lint`在分析前失敗，以繁體中文說明預期與實際版本及官方binary安裝方式，不靜默改用不相容版本。

### 情境三：受限環境無法載入package

開發者先執行`go list ./...`與`golangci-lint run -v`。若Go list成功但lint loader失敗，檢查`GOCACHE`與`GOLANGCI_LINT_CACHE`是否存在且可寫；以環境變數指定該環境允許的cache目錄，不修改repository package或執行無證據的`go mod tidy`。

### 情境四：CLI成功訊息寫入失敗

Policy validation本身成功，但stdout writer返回error。CLI不得回傳0；應嘗試以不含敏感資料的英文訊息寫入stderr並回傳1。

### 情境五：Usage或validation error輸出失敗

輸入本來就應回傳2或1。即使stderr writer失敗，原本的failure exit code仍保持，且程式不得panic或遞迴輸出。

### 情境六：PR lint gate

GitHub Actions使用go.mod指定的Go版本、版本檔指定的golangci-lint官方binary與repository config。Config schema或lint findings任一失敗時，獨立lint job失敗，不授予packages write權限。

## 驗收情境

### 場景 A：Go packages可被標準工具載入

- 測試: `go list ./...`
- 假設: 從repository root使用go.mod指定toolchain
- 當: 列出所有packages
- 那麼: 命令成功且至少包含`vault-agent/cmd/vault-agent`及`vault-agent/internal/syncer/domain`

### 場景 B：v2 config schema有效

- 測試: `golangci-lint config verify -c .golangci.yml`
- 假設: 使用版本檔指定的binary
- 當: 驗證repository config
- 那麼: schema通過，config明確為version 2

### 場景 C：版本不符快速失敗

- 測試: Makefile version guard shell test
- 假設: 注入回報非`2.12.2`的fake或替代binary
- 當: 執行版本檢查target
- 那麼: lint分析不執行，命令非0並顯示預期與實際版本

### 場景 D：CLI success output error

- 測試: `TestRunCLI_ValidatePolicySuccessWriterFailure`
- 假設: policy有效且stdout writer返回sentinel error
- 當: 執行policy validate
- 那麼: exit code為1，stderr只有不含policy值的英文write failure訊息

### 場景 E：CLI failure output error

- 測試: `TestRunCLI_ErrorWriterFailurePreservesExitCode`
- 假設: CLI輸入不合法或policy validation失敗，stderr writer返回error
- 當: 執行CLI
- 那麼: exit code維持2或1，process不panic

### 場景 F：固定設定下lint通過

- 測試: `make lint`
- 假設: PATH提供版本檔指定的官方binary
- 當: 從repository root執行lint
- 那麼: 使用`.golangci.yml`分析全部packages，4個既有`errcheck`已修正，命令返回0

### 場景 G：CI使用同一版本來源

- 測試: workflow static inspection與GitHub Actions run
- 假設: PR或main push觸發lint workflow
- 當: action安裝工具
- 那麼: version來源為`.golangci-lint-version`，config verify與lint均成功，job permissions只有contents read

## 驗收條件

- [x] `go list ./...`成功並可列出專案packages。
- [x] `.golangci.yml`包含`version: "2"`且schema verify通過。
- [x] config明確啟用`errcheck`、`govet`、`ineffassign`、`staticcheck`與`unused`。
- [x] config不依賴home config、不使用`default: all`或全域errcheck exclusion。
- [x] `.golangci-lint-version`固定`v2.12.2`。
- [x] `make lint`驗證binary版本並明確指定repository config。
- [x] policy CLI的4個errcheck findings已修正。
- [x] success output write failure回傳1並有測試。
- [x] usage/validation failure writer error不改變原failure exit code且不panic。
- [x] `make lint`以固定版本返回0。
- [x] 全專案race tests、vet與policy validation tests通過。
- [x] lint workflow使用獨立job／workflow與只讀contents權限。
- [x] local與CI共用同一版本檔，不重複硬編碼兩個版本來源。
- [x] README與中英文開發文件記錄lint安裝、執行與cache排障。
- [x] runtime CLI訊息維持英文且不包含機密值。
- [ ] 實際GitHub Actions lint workflow執行通過。

## 影響範圍

- `.golangci.yml`
- `.golangci-lint-version`
- `Makefile`
- `cmd/vault-agent/policy_command.go`
- `cmd/vault-agent/policy_command_test.go`
- `.github/workflows/golangci-lint.yml`
- `README.md`
- `docs/README.zh-Hant.md`
- `docs/README.en.md`
- 本 spec 文件

## 驗證需求

- 先保存`go list ./...`與`golangci-lint run -v ./...`的baseline，不把環境症狀誤記為程式根因。
- config建立後執行`golangci-lint config verify -c .golangci.yml`與`golangci-lint config path`。
- 使用版本檔指定的官方binary執行`make lint`，不得以現有錯誤版本結果代替。
- CLI writer tests使用可控制的`io.Writer` sentinel error，不依賴磁碟滿、closed pipe或OS-specific行為。
- 執行`go test -race -count=1 ./...`、`go vet ./...`、`make policy-validate`。
- 靜態檢查workflow permissions、version-file與config路徑；CI runtime結果不得由本機測試假造。
- 執行`git diff --check`、YAML parse與Boundary檢查。

## 已知風險

- 固定版本可避免無預警漂移，但需要明確的定期升級流程；不得使用`latest`。
- `v2.12.2`官方binary與本機Homebrew自行build的同版本可能使用不同Go build version；官方建議binary release，本spec以官方binary為準。
- `modules-download-mode: readonly`會在go.mod/go.sum不完整時直接失敗；這是預期品質閘門，不應在lint中自動修改module files。
- 受限執行環境的cache權限不是repository可完全控制；文件只提供可移植的環境變數方式，不硬編碼路徑。
- 修正writer error可能新增CLI failure path；必須以exit code與不洩密測試固定。
- 新增CI lint gate會讓現有未修finding直接阻擋PR，因此4個errcheck必須在同一變更修正後才啟用gate。

## 待確認項目

- 無。工具版本採官方目前patch release`v2.12.2`；linters先固定現有5個，不擴張規則集合。

## 官方參考

- golangci-lint v2 config：<https://golangci-lint.run/docs/configuration/file/>
- golangci-lint local installation：<https://golangci-lint.run/docs/welcome/install/local/>
- golangci-lint CI installation：<https://golangci-lint.run/docs/welcome/install/ci/>
- 官方GitHub Action：<https://github.com/golangci/golangci-lint-action>
