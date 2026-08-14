# Roadmap：ReleaseHub 第一階段

Status: Draft

## 文件定位

- 產品摘要：`brief.md`。
- 完整需求探索：`discovery.md`。
- 本文件只描述產品推進順序，不預先決定正式 spec、設計或 task 的拆分數量。

## 規劃原則

1. 先建立能安全接管 Argo CD Application 的平台基礎。
2. 再完成可使用的發佈申請、審核與部署流程。
3. 最後補齊 rollback、通知與異常保護。
4. 每一階段需求確認後，再決定需要一份或多份正式 `requirements.md`。

## 階段一：平台基礎與 Argo CD 接管

Status: Requirements drafted

目標：讓 ReleaseHub 知道誰能管理哪些 Application，並能安全讀取 ECR 版本及操作 Argo CD。

範圍：

- Organization、Project、Environment、Application 基本階層。
- 永久保留 Organization 的資料模型，單租戶介面隱藏該層。
- Project 自訂 Environment 名稱、數量與環境等級。
- OIDC 登入、群組、角色與權限。
- 單一 Argo CD instance。
- 以 ReleaseHub label 發現 Application，經管理者確認後 onboarding 並關閉 automated sync。
- 專用 Argo CD service account 與最小權限 API token。
- Kustomize image override 對應。
- 單一 AWS account、單一 region 的 ECR tag 查詢與 digest 鎖定。
- Environment 類型固定為 Development、Testing、Staging、Production。
- 發現 label 固定為 `releasehub.io/managed: "true"`，移除時視為設定漂移。
- Application 與部署狀態採固定間隔同步加手動 refresh。

完成判斷：

- 授權使用者能看到自己有權限的 Application。
- Application 能完成 onboarding 並通過設定驗證。
- label 同步不會在未經管理者確認時自動接管 Application。
- ReleaseHub 能從 ECR 選 tag、鎖定 digest，但此階段不必完成正式發佈流程。
- 階段一正式需求已建立，等待審閱；同步間隔等預設值保留為正式需求待確認項目。

## 階段二：發佈申請、審核與部署

Status: Needs clarification

目標：完成 ReleaseHub 的主要價值，讓多 Application release 經過可配置流程後才部署。

範圍：

- release 建立與版本化。
- 手動選擇 Application 或使用 Application Group。
- workflow 樣板、圖形化設定、版本與審核。
- 多 Application 部署、全域並行數、排隊與狀態追蹤。
- `Partial Failed` 與失敗項目 retry。
- Project 與 Environment 操作鎖。

尚待確認：

- workflow 第一階段支援的審核條件。
- release 必填欄位與顯示編號。
- Application Group 與預設順序由誰管理。

完成判斷：

- 未符合 workflow 條件的 release 無法部署。
- 合法 release 能對多個 Application 套用鎖定 digest 並觸發 sync。
- 成功、失敗及部分失敗狀態可正確追蹤。

## 階段三：Rollback、通知與營運保護

Status: Depends on stage two

目標：讓已發佈版本發生問題時可以受控回復，並讓使用者知道設定漂移與異常狀態。

範圍：

- 從成功歷史 release 整批 rollback。
- ECR digest 存在性檢查。
- rollback 權限、二次確認、原因與操作鎖。
- `Rollback Partial Failed` 與失敗項目 retry。
- automated sync 設定漂移阻擋。
- 站內通知、資訊遮蔽與稽核紀錄。

完成判斷：

- 使用者只能 rollback 到仍存在於 ECR 的成功版本。
- rollback 不會與其他部署互相覆蓋。
- 設定漂移會阻擋操作並通知正確對象。

## 整體驗證

1. 以一個 Project、兩個 Environment 及多個 Application 建立測試案例。
2. 驗證無權限使用者無法查看或操作資源。
3. 驗證 release 必須依 workflow 通過後才能部署。
4. 驗證多 Application 部署、部分失敗、retry 與操作鎖。
5. 驗證成功歷史 rollback、ECR image 已刪除阻擋及通知。

## 下一步

1. 名詞定義已確認，作為所有階段的共用需求語言。
2. 繼續收斂階段一需求。
3. 階段一需求確認後，再決定正式 spec 的實際拆分方式。

## Promoted specs

- 階段一：`.spaces/2026-08-13-10-12_Feature-platform-foundation-argocd-onboarding/requirements.md`，Status: Draft。
