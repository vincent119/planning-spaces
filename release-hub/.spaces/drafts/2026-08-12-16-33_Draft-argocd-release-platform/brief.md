# Draft：發佈平台第一階段串接 Argo CD

Status: Draft

## 文件定位

- 規劃對象：ReleaseHub 發佈治理平台。
- 程式碼 repo：`/Users/vincent/Documents/git_home/vin/ReleaseHub`。
- 規劃文件根目錄：`/Users/vincent/Documents/git_home/vin/planning-spaces/release-hub/.spaces`。
- 詳細探索：`discovery.md`。
- 需求拆分與交付順序：`roadmap.md`。
- 本階段只規劃需求，不產出設計、task 或程式碼。

## 產品定位

Argo CD 負責 GitOps reconciliation 與 Kubernetes 部署狀態，但不提供符合團隊需求的發佈申請、可配置審核、整批多 Application 發佈及受控 rollback。ReleaseHub 在 Git 變更與 Argo CD sync 之間提供治理層，避免 `git push` 直接等同發佈完成。

核心流程：

```text
建立 release
→ 套用 workflow
→ 完成必要審核或走直接發佈流程
→ 套用 Kustomize image override
→ 觸發 Argo CD sync
→ 追蹤多個 Application
→ 必要時從成功歷史 release 整批 rollback
```

## 第一階段範圍

1. OIDC、帳號、群組、角色、scope、deny policy 與多租戶資料隔離。
2. `Organization → Project → Environment → Application` 資源模型；單租戶可省略 Organization。
3. 單一 Argo CD instance 與多個受管 Application onboarding。
4. 第一階段只支援 Kustomize image override，不修改 GitOps repo。
5. 單一 ECR 整合、tag 選版及 digest 鎖定。
6. 圖形化、可版本化 workflow 與基礎樣板。
7. 同一 Environment 內多 Application release、審核、部署、重試、狀態彙整與稽核。
8. 從完整成功歷史 release 執行整批 rollback。
9. 站內通知、設定漂移偵測與 Project／Environment 操作鎖。

## 核心決策

- `discovery.md` 的名詞與對應關係已確認，後續需求統一沿用，不再重新定義同名概念。
- 一次 release 限定單一 Organization、Project、Environment，可包含該 Environment 的部分或全部 Application。
- Application 與 Argo CD Application 一對一對應，各 Environment 使用獨立 Argo CD Application。
- 受管 Application 關閉 automated sync；設定被外部覆蓋時阻擋 release、部署與 rollback。
- 使用者以 ECR tag 選版，release 建立時解析並鎖定 digest；部署、retry、rollback 一律使用 digest。
- release 建立後保存 Application、順序、workflow version 與 image digest 快照；內容變更產生新版本並重新審核。
- 多 Application 平行部署，受全平台最大並行數限制，超額項目依 release 順序排隊。
- `Partial Failed` 只重試失敗項目；部署與 rollback 未收斂前鎖定相同 Project 與 Environment。
- rollback 手動觸發、不走 workflow、強制二次確認與原因，只允許整批回到曾 `Succeeded` 的歷史 release。
- 權限採 `User → Group → Role → Scope`，不直接授權個別使用者；本地停權與 deny policy 優先。
- 第一階段不設 `organization_manager`，Organization 層由平台管理者管理。
- 資料模型永遠保留 Organization；單租戶模式只在介面隱藏該層。
- Environment 名稱與數量可由 Project 自訂，並具有供 workflow 與權限套用的類型或等級。
- Application 以 Argo CD label 發現，再由管理者人工確認接管；label 不代表自動啟用管理。
- Argo CD 使用專用 service account 與最小權限 API token，不能建立或刪除 Application，也不能修改 Argo CD Project。
- 第一階段 ECR 限定單一 AWS account、單一 region。
- Environment 名稱可自訂，但類型固定為 Development、Testing、Staging、Production。
- Application 發現 label 固定為 `releasehub.io/managed: "true"`。
- 既有受管 Application 的 label 被移除時，標記設定漂移、阻擋新 release 並通知管理者，不自動解除接管。
- Application 清單與部署狀態採固定間隔同步，並提供手動 refresh。

## 非目標

1. 安裝、升級或取代 Argo CD。
2. 修改 GitOps repo、Kubernetes manifest、Helm chart 或 Kustomize 檔案。
3. 支援 Helm parameter override、其他 OCI registry 或多個 Argo CD instance。
4. 建置 image、CI pipeline 或自行提供 registry。
5. 排程自動部署、Canary、Blue-Green 或 progressive delivery。
6. 電子郵件、Slack、Microsoft Teams 或 Webhook 通知。

## 目前未決重點

1. workflow 第一階段支援的審核條件與節點類型。
2. Release 建立欄位與人類可讀編號。
3. Application Group 與 Project 預設順序的管理權限。
4. Argo CD sync options。
5. ECR 顯示 metadata 與 Application 對應規則細節。
6. 固定同步間隔的預設值與全域設定方式。
7. 第一階段 Web UI、API 與 CLI 的產品介面邊界。

## 下一步

1. 依 `roadmap.md` 先收斂階段一需求。
2. 階段一需求確認後，再決定正式 `requirements.md` 的實際拆分方式。
3. 正式需求確認前不產出 `design.md`、`tasks.md`，也不實作。

## Promoted specs

- 階段一：`.spaces/2026-08-13-10-12_Feature-platform-foundation-argocd-onboarding/requirements.md`，Status: Draft。
