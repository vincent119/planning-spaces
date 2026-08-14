# Discovery：發佈平台第一階段串接 Argo CD

Status: Draft

> 本文件保存詳細需求探索、已確認決策、名詞、風險與未決問題。產品摘要請讀 `brief.md`，需求拆分與交付順序請讀 `roadmap.md`。

## 文件定位

- 規劃對象：ReleaseHub 發佈平台。
- 程式碼 repo：`/Users/vincent/Documents/git_home/vin/ReleaseHub`。
- 規劃文件根目錄：`/Users/vincent/Documents/git_home/vin/planning-spaces/release-hub/.spaces`。
- 需求來源：使用者提出建立發佈平台，第一階段先串接 Argo CD。
- 現況：ReleaseHub 程式碼 repo 尚無提交，沒有可承接的既有功能或契約。
- 本文件只記錄需求探索，不包含技術設計、實作 task 或程式碼變更。

## 背景

團隊需要一個集中管理應用程式發佈的平台。Argo CD 負責 GitOps 同步與部署狀態，但目前的使用方式缺少符合團隊需求的發佈申請與審核機制；若只依賴 Git 變更與自動同步，程式碼完成 `git push` 後便可能直接進入部署流程，無法在發佈前形成明確的人工作業關卡。

ReleaseHub 第一階段需在 Git 變更與 Argo CD sync 之間提供可追溯的發佈治理流程。發佈流程需能像 Jira workflow 一樣配置狀態、轉換條件與可執行角色，而不是固定成單一審核人或固定審核層數。release 只有走完所套用 workflow 的必要審核節點後，ReleaseHub 才能觸發 Argo CD sync。

當發佈結果不符合預期時，平台需允許使用者從歷史版本中選擇 rollback 目標。可選版號應以 ECR 或其他 image registry 內可取得的 image tag 或 digest 為來源，並能對應到 release 中各 Argo CD Application 的部署版本。

因此 ReleaseHub 不是單純呈現 Argo CD 資訊的操作入口；審核、發佈執行、狀態追蹤、稽核與 rollback 都是第一階段的核心需求。實際核准層級與 rollback 方法仍待確認，本文件維持 Draft。

## 初步目標

1. 定義 ReleaseHub 第一階段服務的使用者角色與核心發佈流程。
2. 定義 ReleaseHub 與單一 Argo CD instance 的整合邊界。
3. 讓授權使用者能在 ReleaseHub 建立包含多個 Argo CD Application 的一次發佈，並查看各 Application 的同步、健康及操作結果。
4. 讓授權使用者能從 ReleaseHub 觸發 Argo CD sync，並定義同步操作需要的權限與保護措施。
5. 提供可配置的發佈 workflow，依 workflow 狀態、轉換條件與角色權限控制 release 是否可進入下一階段。
6. 在 release 執行前提供必要審核關卡，未走完 workflow 所要求的審核不得觸發 Argo CD sync。
7. 從 ECR 或其他 image registry 取得可用映像版本，供建立 release 與選擇 rollback 目標。
8. 讓授權使用者能從歷史版本選擇 rollback 目標，並追蹤各 Application 的回復結果。
9. 建立後續可驗收的需求基線，作為正式 `requirements.md` 的來源。

## 初步使用情境

1. 發佈人員查看指定應用程式目前部署在哪些環境，以及各環境的同步與健康狀態。
2. 發佈人員查看目標版本與叢集實際版本是否一致。
3. 管理者使用圖形化編輯器，從平台提供的基礎樣板建立或調整全平台可用的 workflow，定義狀態、允許的轉換與各轉換的可執行角色。
4. 發佈人員提交 release，release 依所套用的 workflow 在不同審核節點流轉。
5. release 走完必要審核節點後，由授權使用者透過 ReleaseHub 對其中多個 Argo CD Application 發起同步。
6. 發佈人員查看整次發佈及各 Application 的同步進度、成功或失敗結果，以及可供排查的錯誤資訊。
7. 發佈結果不符合預期時，授權使用者從 image registry 提供的歷史版本中選擇 rollback 目標，並查看整體與個別 Application 的回復結果。
8. 管理者設定 Argo CD 與 image registry 連線、可操作的 Application 範圍，以及發佈與審核權限。

## 第一階段候選範圍

以下為候選需求，尚未視為已確認：

- 設定單一 Argo CD instance。
- 對受 ReleaseHub 管理的 Argo CD Application 關閉 automated sync，統一由 ReleaseHub 在 workflow 允許後觸發 sync。
- 第一階段使用 Argo CD Kustomize image override 寫入各受管 container 鎖定的 image digest，不修改 GitOps repo；不支援 Helm parameter override。
- Application onboarding 時自動偵測 container、ECR repository 與 Kustomize image 對應，再由管理者確認後啟用。
- 自動偵測失敗時允許管理者手動補充對應，但必須通過 ECR 連線與 Kustomize image override 驗證後才能啟用。
- 建立一次包含多個 Argo CD Application 的 release。
- 以圖形化拖拉方式建立與維護 release workflow，至少能定義狀態、轉換、可執行角色與必要審核節點。
- 提供至少一個可直接使用或複製修改的基礎 workflow 樣板。
- 基礎 workflow 樣板包含 `Draft → Pending Review → Approved → Deploying → Succeeded / Failed → Rolling Back → Rolled Back / Rollback Failed`。
- 對 workflow 進行版本管理；修改後的新版本只適用於新建立的 release。
- 管理 workflow 的草稿、發布與停用生命週期；未發布的 workflow 不得套用到 release。
- 由平台管理者或具備專案權限的人員，將已發布的 workflow 綁定至專案與環境。
- workflow 已被 release 引用後只能停用，不得刪除；未被引用且符合刪除條件者才可刪除。
- 讓 release 依套用的 workflow 提交、核准、拒絕或進入下一個允許狀態。
- 阻擋尚未完成必要 workflow 節點的 release 觸發 Argo CD sync。
- 串接 ECR，取得 repository 及可使用的 image tag 或 digest。
- 讀取 Application 清單與基本資訊。
- 讀取 Application 的 sync status、health status、revision 與 operation status。
- 依專案、環境或 Application 篩選。
- 手動重新整理狀態。
- 在授權與確認機制下觸發多個 Application 的 sync。
- 多個 Application 預設平行觸發部署，實際同時執行數不得超過系統設定的最大並行數。
- 顯示同步結果與 Argo CD 回傳的錯誤。
- 當部分 Application 成功、部分失敗時，release 顯示 `Partial Failed`。
- 從歷史 image 版本選擇 rollback 目標，對已執行的 release 發起 rollback，並顯示各 Application 的回復結果。
- rollback 前確認歷史 digest 仍存在於 ECR；存在時套用該 digest 的 Kustomize image override 並觸發 sync，不使用 Argo CD 原生 history rollback。
- rollback 由具權限使用者手動觸發，不經 release workflow，且必須對 release 中全部 Application 一致執行，不允許只選部分 Application。
- automated sync 設定漂移時阻擋 release、部署與 rollback，並發出通知告知使用者設定已被其他管理管道覆蓋。
- 第一階段提供站內通知；automated sync 設定漂移通知所有 ReleaseHub 使用者，Webhook 群組通知留作後續候選。
- 無專案權限者只看到一般化的設定漂移警示；具專案權限者才能看到專案、環境、Application 與原因等完整資訊。
- 站內通知支援已讀／未讀、通知歷史、全部標記已讀，以及連回問題 Application 的連結。
- rollback 只允許專案管理者或具專案 rollback 角色的使用者執行，且強制二次確認與填寫原因。
- rollback 執行期間鎖定同一專案與環境，不允許另一個部署或 rollback 同時進行。
- 一般部署進入 `Partial Failed` 時同樣鎖定專案與環境，直到 retry、rollback 或管理者終止並完成狀態確認。
- 操作鎖不影響建立與編輯 draft、提交審核及執行審核，只阻擋部署、retry 與 rollback。
- 管理者終止部署或 rollback 並解除鎖定時，必須填寫原因，並保存解除前後各 Application 的實際 digest。
- 串接 OIDC 身分提供者，第一階段需相容 Keycloak 或 AWS Cognito，並由 ReleaseHub 管理帳號狀態、群組、角色與資源權限。
- OIDC 使用者首次登入後若沒有被指派任何群組或權限，預設無法查看任何專案、Application、release 或完整通知資訊。
- 支援使用者跨多個專案、Application、群組並持有不同角色；同一使用者可同時擁有多個角色。
- OIDC 群組只映射至 ReleaseHub 的平台 `viewer` 群組；平台管理者將使用者或群組加入指定專案，再由該專案的 `project_manager` 指派可操作項目與一般專案角色。
- 第一階段允許管理者建立自訂角色與組合細項權限。
- 支援多租戶與單租戶資源模型；多租戶為 `Organization → Project → Environment → Application`，單租戶省略 Organization。
- 每個 dev、staging、production 環境使用各自獨立的 Argo CD Application，Application 與 Argo CD Application 一對一對應。
- 記錄 release 建立、提交、審核、執行及 rollback 的操作人、時間、目標與結果。
- 拒絕 release 時強制填寫理由。
- release 等待審核逾時時維持等待狀態，不自動拒絕或自動通過。
- release 內容變更時建立新的 release 版本號，不直接覆寫既有版本內容。
- release 新版本一律從所套用 workflow 的起點重新審核，不承接舊版本的審核結果。
- 審核節點可指定特定使用者或角色；指定人員無法處理時，管理者可重新指派。
- workflow 可不包含審核節點；`Approved` 後自動部署或等待人工部署，由 workflow 定義。
- `Failed` 後允許 retry 或 rollback，由 workflow 定義。
- 使用者以 ECR image tag 選版，release 建立時解析並鎖定對應 digest；部署、retry 與 rollback 一律使用該 digest。
- ECR image 掃描狀態不影響版本選擇與發佈，不作為阻擋條件。
- `Partial Failed` 後 retry 只重試失敗的 Application，不重複部署已成功的 Application。

## 非目標

以下項目暫不納入第一階段，除非後續需求確認改變範圍：

1. 取代 Argo CD 的 GitOps reconciliation 與部署控制能力。
2. 在 ReleaseHub 內建立或編輯 Kubernetes manifest、Helm chart 或 Kustomize 設定。
3. 建置完整 CI pipeline、映像檔建置或自行提供 artifact registry；第一階段只串接既有 image registry 讀取版本資訊。
4. 自動修改 GitOps repo 的 image tag 或其他設定。
5. 排程發佈、凍結時段與外部變更管理系統整合。
6. Progressive delivery、Canary 或 Blue-Green 發佈。
7. 通知整合、SLA 報表與進階分析。
8. 同時串接或管理多個 Argo CD instance。

## 初步成功條件

1. 使用者可透過 ReleaseHub 建立包含多個已授權 Argo CD Application 的 release。
2. 顯示的同步狀態、健康狀態與 revision 能與 Argo CD 的實際資料對應。
3. 未授權使用者不得觸發同步，且每次同步操作都可追溯。
4. Argo CD 無法連線、認證失敗或操作失敗時，ReleaseHub 能呈現明確狀態，不把失敗顯示為成功。
5. Argo CD 憑證或 token 不得出現在前端、日誌或一般錯誤訊息中。
6. release 未完成所套用 workflow 的必要審核節點時，不得由 ReleaseHub 觸發任何 Application sync。
7. 每次 workflow 狀態轉換必須記錄執行人、時間、來源狀態、目標狀態與理由，且不可因重新整理或服務重啟而遺失。
8. 未具備對應角色或條件不成立的使用者，不得執行受限制的 workflow 狀態轉換。
9. 使用者可查看 image registry 中符合該 Application 的歷史 image tag 或 digest，並從中選擇 rollback 目標。
10. release 與 rollback 必須保存各 Application 選定的 image repository、tag，以及選版當時由 ECR 解析到的 digest。
11. rollback 必須分別呈現每個 Application 的回復結果；部分失敗不得顯示為整體成功。
12. workflow 修改並發布新版本後，進行中的 release 必須繼續使用建立時綁定的 workflow 版本。
13. 拒絕 release 的狀態轉換必須填寫理由，未填寫不得完成拒絕操作。
14. release 等待審核超過任何時間長度時，狀態不得自動改為拒絕或通過。
15. release 內容一旦進入版本紀錄即不可原地覆寫；任何內容變更都必須產生可獨立識別的新版本號，並保留舊版本及其審核紀錄。
16. release 新版本建立後必須從 workflow 起點重新審核，且舊版本的審核結果不得直接沿用。
17. workflow 處於草稿或停用狀態時，不得套用到新的 release；只有已發布 workflow 可被選擇。
18. workflow 已被任何 release 引用後，不得刪除其定義與版本，只能停用以阻止新 release 使用。
19. workflow 的發布、停用、刪除及版本異動都必須留下操作紀錄。
20. workflow 審核節點必須同時支援指定使用者與指定角色。
21. 被指定的審核人無法處理時，管理者必須能重新指派，並保留原指派、新指派、操作者、時間與理由。
22. 未具平台管理或對應專案權限的使用者，不得替專案或 release 選擇 workflow。
23. 第一階段只允許從 ECR 取得 image repository、tag、digest 與所需 metadata。
24. 多個 Application 預設平行觸發，但必須受系統層級最大並行數限制；個別 Application 的結果需獨立追蹤。
25. 部分 Application 成功、部分失敗時，整體 release 狀態必須為 `Partial Failed`，不得顯示為 `Succeeded` 或籠統的 `Failed`。
26. 部署失敗時，ReleaseHub 不得自行假定服務已中斷；應分別呈現新版本部署結果及 Argo CD／Kubernetes 回報的既有 workload 狀態。
27. workflow 可發布不含審核節點的直接發佈流程；平台不強制每個 workflow 都具備審核關卡。
28. workflow 發布前仍需通過基本結構驗證，包括存在起點、可抵達的部署路徑、終止狀態，以及必要轉換具有可執行角色；驗證不得把「沒有審核節點」視為錯誤。
29. `Approved` 後自動進入部署或等待具權限者手動觸發，由 workflow 定義。
30. `Failed` 後進行 retry 或 rollback，由 workflow 定義。
31. ECR image 安全掃描狀態不阻擋選版、審核或發佈。
32. 使用者透過 image tag 選版時，系統必須同時解析並保存當下 digest，以識別後續 tag 是否發生版本漂移。
33. `Partial Failed` 後 retry 只允許重試失敗的 Application；已成功者不得因 retry 被再次部署。
34. 系統必須提供全平台共用的最大部署並行數設定，不允許專案或環境個別覆寫。
35. 超過最大並行數的 Application 必須依 release 內定義的順序排隊；取得執行名額後才可觸發 Argo CD 操作。
36. release 建立時必須將使用者選擇的 tag 解析為 digest 並鎖定；部署、retry 與 rollback 都必須使用同一 digest。

## 假設

1. 假設 Argo CD 已存在且由其他流程管理，ReleaseHub 不負責安裝或升級 Argo CD。
2. 假設 ReleaseHub 透過 Argo CD 正式 API 整合，不直接讀寫 Argo CD 資料庫。
3. 假設 Application 與環境的對應可由 Argo CD project、label、annotation 或 ReleaseHub 設定取得；實際來源尚待確認。
4. 假設 ReleaseHub 負責發佈流程治理，Argo CD 仍負責實際 GitOps reconciliation 與 Kubernetes 資源同步。
5. 假設 `git push` 本身不代表 release 已獲准，也不得被 ReleaseHub 視為發佈完成。
6. 假設 ECR 已存在且由其他系統建置與管理，ReleaseHub 只讀取可用映像版本及必要 metadata。

## 已確認決策

1. 一次發佈的目標為多個 Argo CD Application，不限於單一 Application。
2. ReleaseHub 需要呈現 release 整體狀態及各 Application 的個別狀態。
3. Application 的執行順序、並行方式與部分失敗處理尚未確認，不在本次決策中預先指定。
4. 第一階段只串接單一 Argo CD instance。
5. 第一階段必須允許授權使用者從 ReleaseHub 觸發 Argo CD sync，不是唯讀查詢平台。
6. 第一階段必須包含 release 審核機制；release 未核准前不得觸發 sync。
7. 第一階段必須包含 rollback 機制，並能追蹤多個 Application 的個別回復結果。
8. 審核機制必須支援類似 Jira 的可配置 workflow，不固定為一人或固定人數的審核流程。
9. workflow 必須能定義狀態、允許的狀態轉換、轉換條件及可執行角色。
10. rollback 由使用者從歷史版本中選擇，不固定回到上一個成功 release。
11. release 與 rollback 的可選版本需從 ECR 取得 image tag 與 digest。
12. workflow 由平台層級集中建立與管理，不由每個產品自行維護獨立的 workflow 定義。
13. workflow 第一階段必須提供圖形化拖拉編輯能力，並內建至少一個基礎樣板。
14. dev、staging、production 使用哪個流程，以及申請人是否能自審，均由 workflow 規則控制，不寫死為平台固定行為。
15. 審核人數、通過條件、依序審核與緊急跳關能力均由 workflow 定義；第一階段需支援的條件集合仍待確認。
16. workflow 修改只影響新建立的 release；進行中的 release 繼續使用建立當下綁定的 workflow 版本。
17. 拒絕 release 時必須填寫理由。
18. 審核沒有自動逾時處置；逾時後維持等待狀態。
19. release 內容變更後必須產生新的 release 版本號，既有版本不得被覆寫。
20. release 新版本一律從 workflow 起點重新審核，不承接舊版本的審核結果。
21. workflow 具有草稿、發布與停用狀態；未發布 workflow 不得套用到 release。
22. workflow 由平台管理者或具備對應專案權限的人員選擇，不能由一般 release 建立者任意選用。
23. 已被 release 引用的 workflow 只能停用，不能刪除；未被引用的 workflow 可在符合刪除條件時刪除。
24. 審核節點同時支援指定使用者與指定角色。
25. 指定審核人無法處理時，管理者可以重新指派。
26. 第一階段 image registry 整合範圍只包含 ECR，不包含其他 OCI registry。
27. 基礎 workflow 樣板採用 `Draft → Pending Review → Approved → Deploying → Succeeded / Failed → Rolling Back → Rolled Back / Rollback Failed`。
28. workflow 與專案及環境綁定，不由每次 release 臨時選擇。
29. workflow 允許不包含任何審核節點；此類 workflow 代表直接發佈流程。
30. workflow 發布前需要基本結構驗證，但不強制必須具有審核節點。
31. `Approved` 後自動部署或人工觸發部署，由 workflow 定義。
32. `Failed` 後 retry 或 rollback，由 workflow 定義。
33. 多個 Application 基本上平行部署，最大並行數由系統設定。
34. 部分 Application 成功、部分失敗時，整體狀態為 `Partial Failed`。
35. 使用者以 ECR image tag 選版，系統同時解析並保存選版當時的 digest。
36. ECR image 掃描狀態不作為發佈阻擋條件。
37. `Partial Failed` 後只重試失敗的 Application。
38. 最大並行數為全平台共用設定，專案與環境不得覆寫。
39. 超過最大並行數時，Application 依 release 內順序排隊。
40. 使用者以 tag 選版，但部署、retry 與 rollback 一律使用 release 建立時解析並鎖定的 digest。
41. ReleaseHub 不修改 GitOps repo；版本發布與 rollback 必須透過 Argo CD 支援的非 Git 覆寫能力完成。
42. 目前 Argo CD Application 使用 automated sync，但此設定可配合 ReleaseHub 管理需求調整。
43. ReleaseHub 使用 Argo CD Kustomize image override 寫入鎖定的 image digest，不修改 GitOps repo。
44. 所有受 ReleaseHub 管理的 Application 必須關閉 automated sync，只能在 workflow 允許後由 ReleaseHub 觸發 sync。
45. 每個 Application 可設定多個受管 container；每個受管 container 必須分別選擇 ECR 版本並鎖定 digest。
46. 每個受管 container 必須保存 container name、ECR repository 與 Kustomize image override 的對應關係。
47. 第一階段只支援 Kustomize image override，不支援 Helm parameter override。
48. Application onboarding 採自動偵測加人工確認；管理者確認受管 container、ECR repository 與 Kustomize image 對應後才能啟用。
49. rollback 不使用 Argo CD 原生 history rollback；系統從歷史 release 取得各 container digest，確認 ECR 仍存在後，重新套用 Kustomize image override 並觸發 sync。
50. 建立 release 與實際部署前都必須驗證 Kustomize image override 對應仍有效。
51. 部署前驗證失敗時，原 release 內容與審核紀錄保持不變，release 標記為阻擋且不得執行；修正後需建立新 release 版本並重新審核。
52. 受管 Application 必須在建立 release 與部署前驗證 automated sync 仍為關閉；偵測到設定漂移後是否一律阻擋，仍待確認。
53. Argo CD Web UI 不對一般使用者開放，但系統仍不得只依賴 UI 關閉作為 automated sync 不會被變更的保證。
54. onboarding 自動偵測失敗時，管理者可手動補充 container、ECR repository 與 Kustomize image 對應；連線或 override 驗證未通過時不得啟用。
55. 一般使用者不開放 Argo CD Web UI、CLI 或 API 操作權限；權限與網路控管的具體落實方式留待設計階段。
56. rollback 目標 digest 已被 ECR lifecycle policy 刪除時，該版本不得執行 rollback；使用者只能改選仍存在的歷史版本。
57. 第一階段不提供 ECR lifecycle policy 刪除預警，只在 rollback 前檢查 digest 是否存在。
58. rollback 由具權限使用者手動觸發，不進入 release workflow。
59. 多 Application rollback 必須整批執行，不允許只回復部分 Application。
60. automated sync 被任何管理管道重新開啟時，Application 必須標記為設定漂移，並禁止建立 release、執行部署及執行 rollback，直到設定修正。
61. 偵測 automated sync 設定漂移時，系統必須發出通知，明確告知使用者設定已被其他管理管道覆蓋。
62. rollback 只允許專案管理者與被授予專案 rollback 角色的使用者執行；一般發佈者預設不具 rollback 權限。
63. rollback 執行前必須二次確認，且使用者必須填寫原因；任一條件未完成不得執行。
64. 整批 rollback 中任一 Application 的目標 digest 不存在於 ECR 時，必須阻擋整批 rollback，不得略過該 Application。
65. rollback 部分 Application 失敗時，整體狀態為 `Rollback Partial Failed`，retry 只重試失敗的 Application。
66. rollback 進行期間，同一專案與環境不得開始其他部署或 rollback，避免 Kustomize image override 互相覆蓋。
67. 第一階段通知通道只包含站內通知；電子郵件、Slack、Microsoft Teams 與 Webhook 群組通知不納入第一階段。
68. automated sync 設定漂移的站內通知發送給所有 ReleaseHub 使用者。
69. `Rollback Partial Failed` 時持續鎖定專案與環境，直到失敗項目 retry 成功，或管理者明確終止 rollback。
70. 管理者終止 rollback 後不得立即解除鎖定；必須先確認各 Application 實際 digest，保存版本不一致狀態，才能解除。
71. rollback 目標只允許選擇整體狀態曾為 `Succeeded` 的歷史 release；`Partial Failed` 或 `Failed` release 不得作為 rollback 目標。
72. 一般部署進入 `Partial Failed` 時持續鎖定同一專案與環境，直到失敗項目 retry 成功、執行 rollback，或管理者終止並完成實際 digest 確認。
73. 無專案權限的使用者收到設定漂移通知時，只能看到平台存在漂移的一般化警示，不得看到專案、環境或 Application 識別資訊；具專案權限者可看到完整資訊。
74. 站內通知必須支援已讀／未讀狀態、通知歷史、全部標記已讀，以及在具權限時連回問題 Application。
75. 使用者可將個人通知標記為已讀；底層設定漂移與通知稽核紀錄不得刪除。通知是否允許從個人列表移除仍待確認。
76. 專案與環境被操作鎖定時，仍允許建立 draft release、編輯未送審 draft、提交審核及執行審核；部署、retry 與 rollback 必須阻擋。
77. 管理者終止部署或 rollback 並解除鎖定時，必須填寫原因，並記錄解除前後各 Application 的實際 digest。
78. ReleaseHub 不管理使用者密碼，透過 OIDC 串接 Keycloak 或 AWS Cognito 進行登入與身分識別。
79. ReleaseHub 必須提供帳號狀態、群組、角色與權限管理；OIDC 提供的身分或群組不直接等同可存取 ReleaseHub 資源。
80. OIDC 使用者首次登入後，若尚未在 ReleaseHub 被指派群組、角色或資源權限，必須採預設拒絕，不能查看任何專案、Application、release 或具識別性的通知內容。
81. 一位使用者可屬於多個群組、存取多個專案與 Application，並在不同專案擁有不同角色。
82. 同一使用者可同時擁有多個角色；workflow 可另外限制申請人不得審核自己的 release。
83. 第一階段內建角色與權限組合至少包含：
    - `project_manager`：workflow 管理、release 建立、審核、執行等專案管理權限。
    - `devops_manager`：release 建立、發佈與 rollback 執行權限。
    - `dev_member`：release 建立權限。
    - `devops_member`：發佈與 rollback 執行權限。
    - `viewer`：唯讀檢視權限。
    - 平台管理者：帳號、群組、角色、全域設定與跨專案管理權限。
84. 使用者失去專案權限後，過去通知與 release 歷史中的敏感資訊必須立即依目前權限遮蔽。
85. 通知歷史預設保存七天，且保存天數可由系統全域設定修改。
86. release、workflow、審核、部署、rollback 與權限異動稽核紀錄第一階段不自動刪除，保留政策後續由管理者制定。
87. 使用者只能將站內通知標記為已讀，不得從個人通知列表刪除；底層稽核紀錄亦不得刪除。
88. OIDC 群組只自動映射至 ReleaseHub 平台 `viewer` 群組，不得因外部群組成員資格直接取得任何專案操作權限。
89. 平台管理者負責將使用者或群組加入指定專案並指派 `project_manager`；`project_manager` 再於自己的專案範圍內指派一般角色與可操作項目。
90. `project_manager` 不得新增、移除或授予另一個 `project_manager`；此類專案管理權限只能由平台管理者異動。
91. 第一階段允許管理者建立自訂角色，並由細項權限組成角色；自訂角色不得突破操作者本身可授予的權限範圍。
92. ReleaseHub 本地帳號停用狀態優先於所有 OIDC 群組與手動授權；停用時必須撤銷全部 session，且不得因後續 OIDC 登入重新取得權限，直到本地重新啟用。
93. `devops_member` 保留發佈與 rollback 權限，因其定位為第一線操作人員；第一階段不拆出獨立 `rollback_executor` 內建角色。
94. 使用者在 Keycloak 或 Cognito 被停用或移除後，必須無法繼續進入 ReleaseHub；既有 session 的撤銷時效與技術契約仍待確認。
95. OIDC access token 採短效契約，使用者從 Keycloak 或 Cognito 移除後最長五分鐘內撤銷 ReleaseHub 存取；若身分提供者支援 back-channel logout，應提前終止 session。
96. OIDC 暫時無法連線時，既有使用者可在目前 token 有效期內繼續操作；token 到期後若無法重新驗證，必須拒絕存取，不允許無限離線使用。
97. 自訂角色分為兩種作用域：平台管理者可建立全平台角色；`project_manager` 可在自己的 Project 內建立專案角色。
98. `project_manager` 建立的專案角色只能組合自己有權授予的權限，不得包含 `project_manager` 或平台管理權限。
99. 有效權限採聯集：OIDC `viewer`、手動角色與自訂角色所授予的權限合併生效；ReleaseHub 本地帳號停用優先於全部授權來源。
100. 多租戶模式第一階段不設 `organization_manager`；Organization、Project 與 `project_manager` 由平台管理者設定。
101. 角色由細項操作權限組成，再分派給 ReleaseHub 群組；使用者透過群組取得角色與權限。群組角色的資源作用域仍待確認。
102. 一次 release 只能屬於單一 Organization、Project 與 Environment，但可包含該 Environment 下多個 Application；不得跨環境發佈。
103. ReleaseHub Project 是平台自己的邏輯分組，不與 Argo CD Project 一對一對應；Application 個別對應至 Argo CD Application。
104. 角色分派給群組時支援 Project、Environment 與 Application 三種資源作用域。
105. 專案權限一律透過 `User → Group → Role → Scope` 取得，不允許直接將角色指派給個別使用者。
106. 個別操作權限只能用來組成角色，不得直接授予群組或使用者。
107. ReleaseHub 同時支援平台全域群組與 Organization 群組：全域群組用於平台管理與共用唯讀能力，Organization 群組用於租戶內 Project、Environment 與 Application 權限。
108. 同一群組可在不同資源作用域綁定不同角色，例如 dev 具有 `devops_member`、production 只有 `viewer`。
109. `project_manager` 可以建立 Project 內群組，並將 Organization 既有群組綁定至自己的 Project；不得修改全域群組、其他 Project 群組，或將群組綁定為 `project_manager`。
110. Project scope 的角色自動向下繼承至該 Project 全部 Environment 與 Application；管理介面必須明確顯示受影響範圍。
111. Environment scope 的角色自動向下繼承至該 Environment 全部 Application。
112. 下層資源支援明確拒絕，以覆蓋上層繼承權限；權限判斷優先序為「本地帳號停用 > 明確拒絕 > 角色權限聯集 > 預設拒絕」。
113. Organization 群組可綁定同一 Organization 下多個 Project，但不得跨 Organization；第一階段仍由平台管理者管理此類跨 Project 綁定，不因此新增 `organization_manager`。
114. 未使用且沒有稽核紀錄的群組可以刪除；已有角色綁定或歷史稽核紀錄的群組只能停用，並保留歷史名稱、成員與角色綁定快照。
115. 群組停用後，成員的新操作權限立即撤銷；等待該成員執行的人工操作不再允許其處理，需由具權限管理者重新指派。
116. 群組停用不撤銷已完成的審核與稽核紀錄，也不自動中斷進行中的部署或 rollback。
117. 第一階段不設 `organization_manager`；Organization 群組跨 Project 綁定仍由平台管理者負責。
118. 群組停用導致審核節點無人可處理時，release 維持等待、標記為「需要重新指派」，並以站內通知告知對應 `project_manager`。
119. 群組重新啟用後，停用前的角色與 scope 綁定不得自動恢復；管理者必須重新確認並手動啟用。
120. 明確拒絕採獨立 deny policy，至少包含 Group、Permission、Scope 與 `Effect: Deny`；deny policy 優先於角色權限聯集。
121. release 建立時同時支援建立者手動選擇 Application，以及從 Project 預先定義的固定 Application 群組帶入。
122. release 允許只包含同一 Environment 中的部分 Application。
123. release 建立時保存 Application 清單快照；固定 Application 群組後續變更只影響新 release，不改變既有 release。
124. Application 排隊順序預設使用 Project 設定順序，release 建立者可拖拉調整；送審後修改順序必須產生新 release 版本並重新審核。
125. issue 或變更單連結第一階段為選填，但 workflow 可將其設為特定流程的必填條件。
126. 預計發佈時間第一階段為選填資訊，不觸發排程或自動部署。
127. ECR tag 預設顯示全部；Project 可設定允許版本的正規表示式，例如 production 只允許 semantic version 格式。
128. Environment 名稱與數量可由 Project 自訂；Environment 必須具有供 workflow 與權限規則使用的類型或等級。
129. 資料模型永遠保留 Organization；單租戶模式只在介面隱藏 Organization，不使用另一套資料階層。
130. ReleaseHub 使用專用 Argo CD service account 與 API token，權限限於讀取 Application、更新 Kustomize image override、refresh 與 sync。
131. Argo CD service account 不得建立或刪除 Application，也不得修改 Argo CD Project。
132. Application 透過預先設定的 ReleaseHub label 被發現；ReleaseHub 可定期或手動同步符合 label 的 Application，並列為待接管候選。
133. Application label 只代表允許發現，不代表自動接管；管理者仍須確認 Organization、Project、Environment、Managed Container 與 ECR 對應。
134. onboarding 驗證通過後，由管理者確認 ReleaseHub 主動關閉 automated sync，並保存操作紀錄，Application 才正式成為受管資源。
135. 第一階段 ECR 整合限定單一 AWS account 與單一 AWS region。
136. Environment 名稱可由 Project 自訂，但 Environment 類型固定為 `Development`、`Testing`、`Staging`、`Production`。
137. Application 發現 label 固定為 `releasehub.io/managed: "true"`。
138. 既有受管 Application 的 ReleaseHub label 被移除時，不自動解除接管；Application 標記為設定漂移、阻擋新 release，並以站內通知告知管理者。
139. Application 清單與部署狀態採固定間隔同步，並允許使用者手動 refresh；第一階段不要求事件串流或即時推送。

## 名詞與對應關係

Status: Confirmed

本章名詞已由使用者確認。後續需求文件必須沿用本章定義；若要變更名詞，需明確更新本章及所有引用文件。

### ReleaseHub 資源名詞

| 名詞 | 定義 | 與外部系統的關係 |
|---|---|---|
| Organization | 多租戶模式中的租戶與最高資料隔離邊界 | 不對應 Argo CD Project；單租戶模式省略此層 |
| Project | ReleaseHub 內的產品、系統或服務群組 | 是 ReleaseHub 自有邏輯分組，不與 Argo CD Project 一對一對應 |
| Environment | Project 下的部署環境，例如 dev、staging、production | 每個 Environment 有各自獨立的 Argo CD Application 集合 |
| Application | ReleaseHub 可管理與發佈的最小應用單位 | 與一個 Argo CD Application 一對一對應 |
| Managed Container | Application 中由 ReleaseHub 管理映像版本的 container | 對應一個 ECR repository 與一個 Kustomize image override |
| Application Group | Project 預先定義、可帶入 release 的 Application 集合 | 不建立或修改 Argo CD 資源，只作為 ReleaseHub 選取樣板 |
| Release | 單一 Organization、Project、Environment 中，一次包含一個或多個 Application 的發佈申請與執行紀錄 | 經 workflow 允許後，對多個 Argo CD Application 執行 override 與 sync |
| Release Version | Release 內容的不可變版本；任何內容或順序變更都產生新版本 | 每一版本獨立保存審核、Application 清單與 image digest 快照 |
| Workflow | 控制 release 狀態、轉換、角色、審核與部署觸發條件的流程 | 不等同 Argo CD sync wave 或 hook |
| Deployment | ReleaseHub 套用鎖定的 image digest 並觸發 Argo CD sync 的操作 | 實際 Kubernetes reconciliation 由 Argo CD 執行 |
| Retry | 對部署或 rollback 失敗的 Application 再次執行相同版本操作 | 不重新選版，不重複處理已成功 Application |
| Rollback | 從成功歷史 release 取回 digest，整批重新套用並 sync 的手動操作 | 不使用 Argo CD 原生 history rollback，也不修改 GitOps repo |
| Operation Lock | Project 與 Environment 上避免部署、retry、rollback 互相覆蓋的操作鎖 | 不阻擋 draft 建立、編輯與審核 |

### Argo CD 與 ECR 名詞

| 名詞 | 定義 | ReleaseHub 使用方式 |
|---|---|---|
| Argo CD Instance | ReleaseHub 串接的 Argo CD 服務端 | 第一階段只支援單一 instance |
| Argo CD Project | Argo CD 自身的 Application 權限與目的地分組 | 可作為 Application metadata，但不決定 ReleaseHub Project |
| Argo CD Application | Argo CD 的 declarative application 資源 | 與 ReleaseHub Application 一對一對應 |
| Automated Sync | Argo CD 偵測差異後自動同步的政策 | 受管 Application 必須關閉；重新開啟視為設定漂移 |
| Kustomize Image Override | Argo CD 對 Kustomize image 的非 Git 覆寫設定 | ReleaseHub 用來寫入選定的 image digest |
| Sync | Argo CD 將期望狀態同步至 Kubernetes 的操作 | 只能在 workflow 允許後由 ReleaseHub 觸發 |
| ECR Repository | AWS ECR 中保存 container image 的 repository | 與 Managed Container 建立對應 |
| Image Tag | 使用者可閱讀與選擇的映像版本標籤 | release 建立時用來選版，不作為最終不可變識別 |
| Image Digest | container image 內容的不可變識別 | release 建立時解析並鎖定；部署、retry、rollback 一律使用 digest |

### 權限名詞

| 名詞 | 定義 |
|---|---|
| User | 由 OIDC 識別並在 ReleaseHub 建立本地狀態的使用者 |
| Group | 使用者集合，可為平台全域、Organization 或 Project 作用域 |
| Permission | 最小操作能力，例如查看、建立 release、審核、部署或 rollback |
| Role | 一組 Permission；可為內建角色或自訂角色 |
| Scope | Role 或 deny policy 生效的 Project、Environment 或 Application 範圍 |
| Deny Policy | 對 Group、Permission、Scope 建立的明確拒絕，優先於角色權限聯集 |
| Platform Administrator | 管理平台、Organization、Project、project_manager、全域群組與全域設定的角色 |
| Project Manager | 管理單一 Project 的一般角色、群組、workflow 與成員授權；不能授予 project_manager |

## 資源模型

### 多租戶模式

第一層 `Organization` 代表租戶。資源階層為：

```text
Organization
└── Project
    └── Environment
        └── Application
```

範例：

```text
Organization A
├── ProjectA
│   ├── dev
│   │   ├── APP1
│   │   └── APP2
│   ├── staging
│   │   ├── APP1
│   │   └── APP2
│   └── production
│       ├── APP1
│       └── APP2
└── ProjectB
    └── ...
```

### 單租戶模式

單租戶部署時，Organization 與 Project 視覺層級合併，使用者直接從 Project 操作：

```text
Project
└── Environment
    └── Application
```

### Application 與環境契約

1. `Application` 與 Argo CD Application 一對一對應。
2. dev、staging、production 各自具有獨立的 Argo CD Application，不共用同一 Application。
3. 同名 Application 可存在於不同 Organization、Project 或 Environment，唯一識別不得只依 Application 名稱。
4. release、workflow 綁定、操作鎖、角色與通知權限都必須依完整資源路徑判斷。
5. 一次 release 的範圍固定為單一 Organization、Project 與 Environment，可包含該環境中的多個 Application。
6. ReleaseHub Project 為自身邏輯分組；Argo CD Project 資訊可作為 Application metadata，但不決定 ReleaseHub 的 Project 階層。

## 未決問題

### 產品範圍

1. workflow 第一階段需支援哪些轉換條件，例如指定角色、指定人數、所有人同意或任一人同意？
2. 是否需要允許平台管理者停用某個 Environment 類型？

### Argo CD 整合

1. 是否允許 sync 選項，例如 prune、dry-run、force 或指定 resource？
2. 固定同步間隔的預設值為何，且是否可由平台管理者全域修改？

### Image registry 整合

18. 可選版本除 tag 與協同顯示的 digest 外，是否還要顯示建立時間等 metadata？
19. 如何將 Argo CD Application 對應到一個或多個 ECR repository？
20. release 建立時哪些資料由使用者輸入，哪些由目前 Organization、Project、Environment 與 ECR 選擇自動帶入？
21. release 標題與變更說明是否必填？
22. Release 是否需要自己的人類可讀編號，例如 `REL-2026-000123`？
23. Application Group 由 `project_manager` 建立，還是具 release 建立權限者也能建立？
24. Project 的 Application 預設順序由誰管理？

### 使用者與治理

21. 第一階段有哪些角色，例如申請人、審核人、發佈者、workflow 管理者與平台管理者？
22. 管理者重新指派審核人時是否強制填寫理由？
23. rollback 或緊急發佈是否也強制填寫理由？
24. 錯誤資訊可以顯示到什麼程度，才能兼顧排錯與敏感資訊保護？

### 操作介面

25. 第一階段只提供 Web UI，還是也需要 API 或 CLI？
26. 使用者最先需要的頁面是待審核清單、release 詳情、環境總覽，還是 Application 清單？

## 拆分判斷

- 目前維持單一 Draft，先確認產品邊界。
- 正式規劃時，預期至少拆成「Argo CD 連線與狀態查詢」、「可配置 workflow 與權限」、「release 執行與稽核」、「image registry 版本目錄」及「rollback 流程」等可獨立驗收的需求範圍。
- 因一次發佈包含多個 Application，各正式需求還需共同定義 release 整體狀態、Application 個別狀態與部分失敗語意。

## 下一個需求確認點

請先確認下列三項，確認後再補齊正式需求：

1. release 標題與變更說明是否必填？
2. Release 是否需要人類可讀編號，例如 `REL-2026-000123`？
3. Application Group 與 Project 預設 Application 順序由哪些角色管理？

## Promoted specs

- 尚未建立。
