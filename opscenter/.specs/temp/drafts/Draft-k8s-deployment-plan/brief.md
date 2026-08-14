# Draft: K8s Deployment Plan

Status: Draft

## Source

- `.kiro/specs/2026-07-14_k8s-deployment-plan`

## Intent

重新規劃 Kubernetes 部署、設定、secret、migration、health check、rollout 與 rollback。

## Bounded Context

包含：

- Deployment manifests
- ConfigMap and Secret
- Service and ingress
- Health check 與 rollout 策略

不包含：

- CI/CD pipeline 完整平台
- 雲端基礎設施建立

## Promotion Notes

Promote 時應使用 `Chore`，驗收需包含 dry-run、rollback 程序與 secret 不落版控。

