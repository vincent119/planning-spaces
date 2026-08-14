# Import Ticket Tools Task

- [x] 0.1 將需求調整為 CLI 匯入工具，不建立前端 UI
- [x] 0.2 將設計調整為 `cmd/import_ticket_data`，保留後續 API 化延伸
- [x] 1.1 建立 `cmd/import_ticket_data` CLI 入口
- [x] 1.2 實作 `.xlsx` 分頁掃描與欄位解析
- [x] 1.3 實作 Ticket 主資料對應與逐列驗證
- [x] 1.4 實作 dry-run 預覽與 `-commit` 寫入
- [x] 1.5 驗證 CLI 編譯與基本測試

- [x] 2.1 補強 CLI 參數：`-auto-create-sub-projects`、`-auto-create-resources`、`-resource-type`、`-save-source-file`
- [x] 2.2 預覽階段標示缺少子專案時的 `will_create_sub_project`
- [x] 2.3 預覽階段標示缺少資訊來源時的 `will_create_resource`
- [x] 2.4 提交時自動建立缺少的 `sub_projects`，並符合後端 key 約束
- [x] 2.5 提交時自動建立缺少的 `ticket_resources`，預設 `is_active=true`
- [x] 2.6 提交時保存原始 Excel 檔，並在 CLI 輸出保存結果
- [x] 2.7 調整描述合併格式：`處理方式`、`備註`、`填寫人`、來源分頁、Excel 列號、Jira 單號
- [x] 2.8 確認 `填寫人` 不查詢、不建立、不對應 `ops_user`
  - 注意：此項僅指不對應專案維運人員表 `ops_user`；後續 `3.1` 會改為對應系統登入使用者表 `users` 以寫入 `tickets.created_by`
- [x] 2.9 驗證 dry-run 不寫入主資料、不保存原始 Excel 檔
- [ ] 2.10 驗證 commit 可建立 Ticket、缺少的子專案、缺少的資訊來源並保存來源檔

- [x] 3.1 修正匯入工具 `tickets.created_by` 對應
  - 匯入時以 Excel「填寫人」查詢 `users.username` 或 `users.full_name`
  - 支援「早-」、「中-」、「晚-」等班別前綴後再查人名
  - 找不到使用者時列為該 row 錯誤，不得默默寫入 admin
  - `-actor-username` 僅作為缺少填寫人時的 fallback 與 CLI 操作者，不得覆蓋已填寫的人員
  - dry-run 預覽需顯示解析後的建立者，方便匯入前檢查
  - 完成：已新增 row-level creator 解析；有 `填寫人` 時查 `users.username/full_name`，空值才 fallback 到 `-actor-username`；`tickets.created_by` 與初始 `ticket_activities.actor_id` 都使用該列建立者

- [x] 3.2 補匯入工具測試與驗證
  - 補使用者名稱正規化測試
  - 補找不到使用者時 row error 測試
  - 補 `created_by` 與初始 `ticket_activities.actor_id` 寫入 row 建立者的測試或可驗證查詢
  - 完成：已補候選人正規化、填寫人查詢、空填寫人 fallback、找不到填寫人 row error、INSERT 使用 row 建立者測試
  - 驗證：`go test ./cmd/import_ticket_data`、`go test ./...` 通過

- [x] 4.1 調整可選欄位驗證與測試
  - 「填寫Jira項目選擇」與「Jira單號」留白時不得產生必填欄位錯誤
  - Jira 單號有值時維持既有重複資料檢查；留白時不進行該檢查
  - 補正規化測試，確認兩欄留白仍可通過欄位必填驗證

- [x] 4.2 支援兩種開單記錄分頁命名與測試
  - 動態掃描同時接受「開單記錄-數字」與「開單紀錄-數字」格式
  - 排除不符合格式的分頁，並維持既有名稱排序

- [x] 4.3 修正未指定子專案的 Ticket 寫入
  - `tickets.sub_project_id` 支援 NULL，與可選 Excel 欄位規則一致
  - 匯入列未指定子專案時，明確寫入 SQL NULL，不得以空字串觸發外鍵錯誤
  - 補寫入測試，驗證空白子專案會傳入 NULL

- [x] 4.4 修正未指定子專案的 Ticket 列表讀取
  - Ticket 列表與詳情以左連接讀取子專案，不得排除 `sub_project_id` 為 NULL 的匯入資料
  - 子專案欄位的 NULL 統一回傳空字串
  - 補查詢 SQL 測試，防止恢復為內連接
