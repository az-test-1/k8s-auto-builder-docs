# ADR-M011: 고가용성 전략 (HPA/PDB)

> **Status**: 승인됨
> **Date**: 2026-01
> **Category**: Migration - High Availability

---

## 상태
✅ **승인됨** (2026-01)

## 컨텍스트
프로덕션 환경에서 서비스 안정성 확보를 위한 자동 스케일링 및 중단 보호 전략 필요

## 결정
**HPA (Horizontal Pod Autoscaler) + PDB (Pod Disruption Budget) 적용**

## HPA 설정

| 대상 | Min | Max | CPU Target | 근거 |
|------|:---:|:---:|:----------:|------|
| cms-web-api | 1 | 5 | 70% | API 서버, 트래픽 변동 대응 |
| cms-celery-worker | 1 | 3 | 60% | 빌드 작업 부하에 따라 확장 |
| cms-web-ui | 1 | 3 | 70% | 프론트엔드, 보통 안정적 |

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cms-web-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cms-web-api
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## PDB 설정

| 대상 | minAvailable | 근거 |
|------|:------------:|------|
| cms-web-api | 1 | 최소 1개 Pod 항상 가동 |
| cms-celery-worker | 1 | 진행 중인 작업 보호 |
| redis | 1 | 데이터 손실 방지 |

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cms-web-api-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: cms-web-api
```

## 관련 파일
```
kubernetes/base/cms/hpa.yaml
kubernetes/base/cms/pdb.yaml
kubernetes/base/redis/pdb.yaml
```

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
