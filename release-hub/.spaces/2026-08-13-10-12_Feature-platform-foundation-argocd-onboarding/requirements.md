# 需求文件：平台基礎與 Argo CD Application 接管

## 來源

- Draft: `.spaces/drafts/2026-08-12-16-33_Draft-argocd-release-platform/`
- Roadmap: 階段一「平台基礎與 Argo CD 接管」
- Type: Feature
- Owner: 待確認
- Status: Draft

## 文件定位

本 spec 將 ReleaseHub roadmap 階段一提升為正式需求，處理資源階層、OIDC 身分、群組與角色授權、單一 Argo CD instance、Application 發現與接管，以及單一 ECR account／region 的版本目錄。

本 spec 不處理 release 建立、workflow、審核、正式部署編排、retry、rollback 或通知中心。這些能力由 roadmap 階段二與階段三接續。本 spec 只需讓受權管理者完成 Application onboarding，並證明 ReleaseHub 能安全讀取 Application 與 ECR 版本資料。

參考來源：

- 需求來源：使用者需求探索與確認。
- 既有文件：來源 Draft 的 `brief.md`、`discovery.md`、`roadmap.md`。
- 既有程式碼：ReleaseHub repo 尚無提交，沒有既有模組可承接。

## 背景

Argo CD 已管理 Kubernetes Application，但目前缺少 ReleaseHub 所需的租戶資源模型、企業身分授權、受管 Application 接管邊界及 ECR 版本目錄。在後續建立 release workflow 前，平台必須先回答三件事：使用者可以看到什麼、哪些 Argo CD Application 由 ReleaseHub 管理，以及每個受管 container 可使用哪些不可變 image digest。

## 問題陳述

目前沒有一致方式將 Organization、Project、Environment 與 Argo CD Application 建立安全對應，也沒有可驗證的 OIDC／RBAC、Application 接管或 ECR digest 鎖定契約。若直接進入發佈功能，可能造成跨租戶資料外洩、未授權操作或使用錯誤 image 版本。

## 目標

1. 建立單租戶與多租戶共用的資源階層及資料隔離規則。
2. 透過 OIDC、群組、角色、scope 與 deny policy 實施預設拒絕的授權模型。
3. 以固定 Argo CD label 發現 Application，並經人工確認後安全接管。
4. 使用最小權限 Argo CD service account 讀取與驗證 Application 設定。
5. 建立 Managed Container、ECR repository 與 Kustomize image override 的可信任對應。
6. 從單一 AWS account、單一 region 的 ECR 查詢 tag 並解析不可變 digest。
7. 偵測 automated sync 或管理 label 的設定漂移並阻止後續 release 使用不可信任 Application。

## 非目標

1. 建立 release、workflow、審核或正式部署執行流程。
2. 執行 rollback、retry、排程部署或多 Application orchestration。
3. 修改 GitOps repo、manifest、Helm chart 或 Kustomize 檔案。
4. 支援 Helm parameter override。
5. 支援多個 Argo CD instance。
6. 支援多 AWS account、多 region 或 ECR 以外的 OCI registry。
7. 自動建立、刪除 Argo CD Application 或修改 Argo CD Project。
8. 以事件串流提供即時 Application 狀態。
9. 安裝或升級 Argo CD、Keycloak、AWS Cognito 或 ECR。

## 已定決策

### 資源模型

- 資料模型固定保留 `Organization → Project → Environment → Application`。
- Organization 是租戶及最高資料隔離邊界；單租戶模式只在 UI 隱藏此層。
- Project 是 ReleaseHub 自有邏輯分組，不與 Argo CD Project 一對一對應。
- Environment 名稱與數量可自訂，類型固定為 `Development`、`Testing`、`Staging`、`Production`。
- ReleaseHub Application 與 Argo CD Application 一對一對應，各 Environment 使用獨立 Argo CD Application。
- 同名 Application 可存在於不同資源路徑，唯一識別不得只使用名稱。

### 身分與權限

- 使用 OIDC 串接 Keycloak 或 AWS Cognito，不由 ReleaseHub 管理密碼。
- OIDC 群組只映射平台 `viewer`；未取得 ReleaseHub 群組與資源授權者看不到任何受保護資源。
- 專案授權固定使用 `User → Group → Role → Scope`，不直接把角色授予個別使用者。
- Scope 支援 Project、Environment、Application，並向下繼承。
- Permission 只能組成 Role，不能直接授予 Group 或 User。
- 有效權限為角色聯集；優先序為本地帳號停用、deny policy、角色權限聯集、預設拒絕。
- 支援平台與 Project 自訂角色；`project_manager` 不能授予 `project_manager` 或平台權限。
- 第一階段不設 `organization_manager`，Organization 層由平台管理者管理。
- ReleaseHub 本地停權立即撤銷 session，且優先於 OIDC 授權。
- OIDC access token 最長五分鐘；若支援 back-channel logout，應提前撤銷 session。

### Argo CD 接管

- 第一階段只串接單一 Argo CD instance。
- ReleaseHub 使用專用 service account 與 API token。
- Argo CD 權限只包含讀取 Application、更新 Kustomize image override、refresh 與 sync；本 spec 不執行正式部署 sync。
- Application 發現 label 固定為 `releasehub.io/managed: "true"`。
- label 只代表允許發現，不代表自動接管。
- ReleaseHub 可固定間隔同步候選 Application，並提供手動 refresh。
- onboarding 採自動偵測加人工確認；偵測失敗時可手動補充，但驗證通過前不得啟用。
- 接管確認後由 ReleaseHub 關閉 automated sync，並保存稽核紀錄。
- 受管 Application 的 label 被移除或 automated sync 被重新開啟時，標記設定漂移並禁止新 release 使用，不自動解除接管。

### ECR 與版本

- 第一階段只支援單一 AWS account、單一 region。
- 每個 Managed Container 對應 container name、ECR repository 與 Kustomize image override。
- 使用者以 tag 選版，ReleaseHub 同時解析並保存 digest。
- digest 是後續部署、retry 與 rollback 的不可變版本識別。
- ECR 掃描狀態不阻擋版本瀏覽或選擇。

## 待確認項目

- Application 與部署狀態固定同步間隔的預設值及全域設定範圍。
- Application 狀態 refresh 的逾時與失敗呈現文字。
- ECR 版本清單除 tag、digest 外，第一階段是否顯示 push time 等 metadata。
- Argo CD sync options 屬於階段二；本 spec 只驗證 service account 權限，不定義正式部署參數。

## 現有行為

ReleaseHub repo 尚無提交，也沒有既有 UI、API、帳號、資源階層、Argo CD 或 ECR 整合。Argo CD Application 已存在並使用 automated sync，且可配合 ReleaseHub 接管需求調整。

## 新行為

1. 使用者透過 OIDC 登入；未授權使用者只能看到空白或無權限狀態。
2. 平台管理者可建立 Organization、Project、Environment，並設定群組、角色及 scope。
3. 管理者可同步具有 ReleaseHub label 的 Argo CD Application 候選清單。
4. 管理者可確認 Application 的資源歸屬及 Managed Container 對應。
5. 驗證通過並確認接管後，ReleaseHub 關閉 automated sync 並將 Application 標記為受管。
6. 授權使用者可查看受管 Application 的同步與健康狀態。
7. 授權使用者可查看 ECR tag；系統同時解析並保存 digest。
8. label、automated sync、override 對應或 ECR 連線漂移時，平台清楚顯示錯誤並阻止不可信任資源進入後續 release。

## 影響範圍

- 使用者：平台管理者、project_manager、devops_manager、dev_member、devops_member、viewer 及自訂角色使用者。
- 功能：登入、帳號狀態、群組、角色、scope、資源瀏覽、Application onboarding、Argo CD 狀態、ECR 版本目錄。
- API / CLI：需提供支援 UI 的內部 API；第一階段不承諾公開 CLI。
- Data / Storage：Organization、Project、Environment、Application、Managed Container、群組、角色、permission、scope、deny policy、連線設定與稽核紀錄。
- 文件 / 安裝 / 發布：需說明 OIDC、Argo CD service account、Application label、ECR IAM 與單／多租戶設定。

## 使用情境

- 作為平台管理者，我想建立租戶與 Project 資源階層，以便隔離不同租戶與團隊資料。
- 作為平台或 Project 管理者，我想透過群組、角色及 scope 授權，以便使用者只存取被允許的資源。
- 作為 Project 管理者，我想從 Argo CD 發現標記過的 Application 並確認接管，以便避免未經確認就改動既有部署設定。
- 作為 DevOps 操作人員，我想查看受管 Application 狀態與 ECR 版本，以便為後續 release 選擇可信任版本。

## 驗收情境

### 情境：未授權 OIDC 使用者採預設拒絕

- 場景：新 OIDC 使用者尚未被加入任何 ReleaseHub 群組或 scope。
- 測試：`待建立：auth_default_deny`
- 假設：使用者已由 Keycloak 或 Cognito 完成登入。
- 當：使用者進入 ReleaseHub 或查詢受保護資源。
- 那麼：不得看到任何 Organization、Project、Environment、Application 或具識別性的通知資料。

### 情境：資源 scope 向下繼承並套用明確拒絕

- 場景：群組在 Project scope 具有 viewer，在 production Environment 對特定 permission 設有 deny policy。
- 測試：`待建立：rbac_scope_and_deny_precedence`
- 假設：使用者屬於該群組。
- 當：使用者存取 dev 與 production 下的 Application。
- 那麼：dev 依 Project 角色授權，production 的明確拒絕優先於角色聯集。

### 情境：單租戶 UI 隱藏 Organization 但資料仍隔離

- 場景：系統以單租戶模式執行。
- 測試：`待建立：single_tenant_hidden_organization`
- 假設：資料模型已建立預設 Organization。
- 當：使用者瀏覽 Project 與建立 Environment。
- 那麼：UI 不要求選擇 Organization，但所有資源仍保存 Organization 識別。

### 情境：自訂 Environment 名稱使用固定類型

- 場景：Project 管理者建立名稱為 `uat-tw` 的 Environment。
- 測試：`待建立：custom_environment_fixed_type`
- 假設：使用者具對應管理權限。
- 當：選擇 `Testing` 類型並儲存。
- 那麼：Environment 建立成功，名稱為 `uat-tw`，類型為固定集合中的 `Testing`。

### 情境：label 發現不會自動接管

- 場景：Argo CD Application 具有 `releasehub.io/managed: "true"`。
- 測試：`待建立：argocd_label_discovery_requires_confirmation`
- 假設：ReleaseHub 能連線 Argo CD。
- 當：固定同步或管理者手動 refresh。
- 那麼：Application 出現在待接管清單，但 automated sync 與 Application 設定不得被修改。

### 情境：確認接管 Application

- 場景：管理者確認候選 Application 的 Organization、Project、Environment、Managed Container 與 ECR 對應。
- 測試：`待建立：argocd_application_onboarding`
- 假設：Kustomize image override、ECR repository 與權限驗證皆通過。
- 當：管理者確認接管。
- 那麼：ReleaseHub 關閉 automated sync、保存操作稽核，並將 Application 標記為受管。

### 情境：接管驗證失敗

- 場景：自動偵測不到 Kustomize image 對應，或管理者手動設定錯誤 repository。
- 測試：`待建立：onboarding_validation_failure`
- 假設：Application 尚未正式受管。
- 當：管理者執行驗證或確認接管。
- 那麼：系統顯示可理解的錯誤，不得關閉 automated sync，也不得將 Application 標記為受管。

### 情境：偵測受管 Application 設定漂移

- 場景：受管 Application 的 label 被移除，或 automated sync 被外部重新開啟。
- 測試：`待建立：managed_application_drift`
- 假設：Application 已完成 onboarding。
- 當：固定同步、手動 refresh 或建立後續 release 前驗證。
- 那麼：Application 標記為設定漂移、不自動解除接管，並禁止新 release 使用。

### 情境：查詢 ECR tag 並鎖定 digest

- 場景：受管 container 已對應單一 account／region 的 ECR repository。
- 測試：`待建立：ecr_tag_resolves_digest`
- 假設：使用者具查看權限，ECR image 存在。
- 當：使用者選擇 tag。
- 那麼：ReleaseHub 顯示 tag 並解析、保存對應 digest，且不得只保存 tag。

### 情境：跨 Organization 存取被隔離

- 場景：相同 Project 或 Application 名稱存在於兩個 Organization。
- 測試：`待建立：tenant_resource_isolation`
- 假設：使用者只被授權其中一個 Organization 的資源。
- 當：使用者瀏覽、搜尋或直接請求另一 Organization 的資源識別。
- 那麼：不得回傳另一 Organization 的資料，也不得透過名稱衝突誤取資源。

## 驗收條件

1. 上述所有驗收情境具有可執行測試或正式設計階段確認的測試 selector。
2. 未授權 OIDC 使用者無法讀取任何租戶受保護資料。
3. 單租戶與多租戶使用同一 Organization 資料模型。
4. Environment 名稱可自訂，但類型只能使用四種固定值。
5. 只有具有固定 label 的 Application 會進入待接管清單。
6. 未經人工確認不得修改 Application 的 automated sync 或受管狀態。
7. Argo CD service account 不具建立、刪除 Application 或修改 Argo CD Project 的權限。
8. onboarding 失敗不得留下部分接管狀態。
9. 受管 Application 發生 label 或 automated sync 漂移時，禁止後續 release 使用。
10. ECR tag 選擇必須同時保存 digest。
11. 不得讀取設定範圍外的 AWS account、region 或 repository。
12. 所有帳號、群組、角色、deny policy、onboarding 與 automated sync 異動都有不可刪除的稽核紀錄。

## 驗證需求

- Unit / Integration：待建立 OIDC、RBAC、tenant isolation、Argo CD onboarding、drift detection 與 ECR integration 測試。
- CLI / Dry-run：無公開 CLI；需提供 Argo CD 與 ECR 連線測試及 onboarding validation dry-run 能力。
- 文件檢查：OIDC、Argo CD RBAC、service account、label、ECR IAM、Environment 類型與單／多租戶設定文件。
- 回歸驗證：此為新專案，沒有既有行為；後續階段不得繞過本 spec 的資源隔離、授權與 onboarding 契約。

## 風險與假設

| 類型 | 內容 | 處理方式 |
|---|---|---|
| 風險 | Argo CD service account 權限過大 | 以允許清單驗證最小權限，拒絕建立、刪除與 Project 修改能力 |
| 風險 | ApplicationSet 或外部管理管道覆寫 automated sync 或 label | 固定同步與操作前驗證設定漂移，阻止後續 release 使用 |
| 風險 | OIDC token 在外部停權後短時間仍有效 | token 最長五分鐘並優先使用 back-channel logout；ReleaseHub 本地停權立即生效 |
| 風險 | Kustomize 自動偵測錯誤 | onboarding 必須人工確認並支援 validation dry-run |
| 風險 | 同名資源造成跨租戶讀取 | 所有查詢使用完整資源識別並執行 Organization scope 檢查 |
| 假設 | Argo CD、ECR 與 OIDC provider 已存在 | ReleaseHub 只整合，不負責安裝或生命週期管理 |
| 假設 | 第一階段只有單一 AWS account、region 與 Argo CD instance | 超出範圍時另立需求，不在本 spec 擴張 |

## 摘要

- 關鍵決策：以單一正式 spec 定義階段一平台基礎、OIDC／RBAC、label 發現、人工接管、Kustomize override 對應及 ECR digest 目錄。
- 待確認項目：固定同步間隔、refresh 錯誤呈現及 ECR 額外 metadata。
- 風險：外部設定漂移、過大權限、OIDC session 撤銷延遲及跨租戶資料隔離。
- 下一步：等待使用者審閱本 `requirements.md`；未經明確指示不建立 `design.md`、`tasks.md` 或實作。
