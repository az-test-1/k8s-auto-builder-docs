# ADR-M013: 로깅 및 모니터링

> **Status**: 진행 중
> **Date**: 2026-01
> **Category**: Migration - Observability

---

## 상태
🔄 **진행 중** (향후 개선 예정)

## 컨텍스트
K8s 환경에서의 로깅/모니터링 전략 필요

## 현재 상태 (MVP)
- **로깅**: kubectl logs로 직접 확인
- **모니터링**: kubectl get pods/hpa로 상태 확인

## 향후 계획

| 구성요소 | 계획 |
|----------|------|
| **로그 수집** | Azure Monitor Container Insights 또는 Loki |
| **메트릭** | Prometheus + Grafana |
| **알림** | Azure Monitor Alerts 또는 Alertmanager |
| **대시보드** | Grafana 대시보드 |

## 현재 구현
```bash
# 로그 확인
kubectl logs -f deployment/cms-web-api -n prd

# 상태 확인
kubectl get pods -n prd
kubectl get hpa -n prd
```

## 개선 방향
1. **Phase 1**: Azure Monitor Container Insights 활성화
2. **Phase 2**: Prometheus + Grafana 스택 배포
3. **Phase 3**: 커스텀 메트릭 대시보드 구성
4. **Phase 4**: 알림 규칙 설정

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
