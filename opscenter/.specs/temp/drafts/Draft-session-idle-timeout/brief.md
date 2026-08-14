# Draft: Session Idle Timeout

Status: Draft

## Source

- 近期「沒有閑置時間自動登出問題」文件與實作

## Intent

用新 SDD 補齊閒置時間自動登出的需求、設計、task 與驗收，讓 session policy API、前端 idle bridge 與登出流程可追溯。

## Bounded Context

包含：

- `/auth/session-policy`
- `security.session.timeout`
- 前端 idle timer
- 使用者互動重置與自動 logout

不包含：

- SSO protocol
- Token rotation 全面重寫

## Promotion Notes

Promote 時需保留已知測試缺口：前端 fake timer 單元測試若尚未建立，必須在 `tasks.md` 標成未完成驗收。

