# Draft: Metrics Observability

Status: Draft

## Source

- `.kiro/specs/2026-07-07_10-07_metrics-observability`

## Intent

重新規劃技術 metrics、health endpoint、Prometheus scraping、structured logging 與 tracing。

## Bounded Context

包含：

- HTTP metrics
- DB metrics
- Health and readiness
- Logger contract

不包含：

- 業務指標治理
- K8s manifest 完整部署

## Promotion Notes

Promote 時需把 structured access logging 的 bugfix trace 納入已知契約，避免再次混用 Gin 預設 log。

