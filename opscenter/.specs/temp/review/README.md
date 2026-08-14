# 舊設計改用新版 SDD 的重新規劃審閱稿

Status: Draft

## 文件定位

本目錄用來審閱：

「既有 `.kiro/specs` 的舊設計，如果改用目前 `sdd-skill` 從頭規劃，會如何從 Draft 開始建立，並對應到每個功能？」

本目錄不取代舊 spec，不搬移原始文件，也不代表已經 promote 成正式 spec。

## 文件清單

- `00-replan-method.md`：新版 SDD 重新規劃的方法與拆分原則。
- `01-feature-draft-map.md`：每個舊功能對應的新 Draft 規劃。
- `02-promotion-shape.md`：Draft promote 成正式 spec 時應長什麼樣子。
- `03-review-notes.md`：目前看起來怪的地方、修正後的判斷與待確認事項。

## 結論

舊設計若改用新版 SDD，不應直接把每個舊目錄搬成新目錄。正確做法是：

1. 先建立 Draft，確認功能邊界與依賴。
2. 每個使用者可感知功能維持一個 Draft。
3. 共用契約先抽出，再讓 Ticket、Dashboard、SLA、Report 等功能引用。
4. 只有確認要實作或補正式文件時，才 promote 成 `Feature`、`BugFix`、`Refactor`、`Docs` 或 `Chore` spec。

