# ADR-005: 멀티 클러스터 배포 전략

## 상태
채택됨 (Accepted)

## 날짜
2026-01-12

## 컨텍스트

Auto-Builder 서비스는 다음과 같은 환경 분리가 필요하다:

- **운영 환경 (PRD)**: 안정성과 격리가 최우선
- **비운영 환경**: dev, stage, sandbox, test 등 다양한 환경 필요

### 고려 사항
1. 운영 환경은 개발/테스트 환경과 완전히 격리되어야 함
2. 비운영 환경은 비용 효율성을 위해 리소스 공유 가능
3. ArgoCD를 통한 GitOps 배포 일원화

## 고려한 대안

### 대안 1: 단일 클러스터 + Namespace 분리
```
AKS Cluster (1개)
├── namespace: prd
├── namespace: dev
├── namespace: stage
└── namespace: test
```
- 장점: 비용 최소화, 관리 단순
- 단점: 운영/개발 격리 불가, 리소스 경합 위험

### 대안 2: 환경별 클러스터 분리 (채택)
```
PRD Cluster (운영 전용)
└── namespace: prd

DEV Cluster (비운영 공용)
├── namespace: dev
├── namespace: stage
├── namespace: sandbox
└── namespace: test
```
- 장점: 운영 완전 격리, 비운영은 비용 효율적
- 단점: 클러스터 2개 운영 필요

### 대안 3: 환경별 완전 분리
```
PRD Cluster → namespace: prd
DEV Cluster → namespace: dev
Stage Cluster → namespace: stage
Test Cluster → namespace: test
```
- 장점: 완전 격리
- 단점: 비용 과다, 관리 복잡

## 결정

**대안 2: PRD/DEV 클러스터 분리**를 채택한다.

### 클러스터 구성

```
┌─────────────────────────────────────┐
│  aks-az01-prd-ab-01 (운영 클러스터)  │
│  ├── ArgoCD 설치 (허브)              │
│  └── namespace: prd                 │
└─────────────────────────────────────┘
              │
              │ ArgoCD가 두 클러스터 모두 관리
              │
┌─────────────────────────────────────┐
│  aks-az01-dev-ab-01 (개발 클러스터)  │
│  ├── namespace: dev                 │
│  ├── namespace: stage               │
│  ├── namespace: sandbox             │
│  └── namespace: test                │
└─────────────────────────────────────┘
```

### ArgoCD 멀티 클러스터 설정

1. **PRD 클러스터에 ArgoCD 설치** (허브 역할)
2. **DEV 클러스터를 ArgoCD에 등록**
   ```bash
   argocd cluster add dev-cluster --name dev-cluster
   ```
3. **ApplicationSet에서 클러스터 URL 사용**
   ```yaml
   generators:
     - list:
         elements:
           - env: prd
             clusterUrl: https://kubernetes.default.svc
           - env: dev
             clusterUrl: https://dev-cluster-api-server-url
   ```

### 환경 추가 절차

DEV 클러스터에 새 환경(예: stage) 추가 시:

1. ApplicationSet에 환경 추가:
   ```yaml
   elements:
     - env: stage
       cluster: dev-cluster
       clusterUrl: https://dev-cluster-api-server-url
   ```

2. Overlay 폴더 생성:
   ```bash
   cp -r k8s/overlays/dev k8s/overlays/stage
   ```

3. Secret 생성 (DEV 클러스터에서):
   ```bash
   kubectl create namespace stage
   kubectl create secret generic app-secrets -n stage ...
   ```

## 결과

### 장점
- 운영 환경 완전 격리 (장애 전파 방지)
- 비운영 환경 비용 효율적 (클러스터 1개로 다중 환경)
- ArgoCD 단일 포인트에서 전체 환경 관리
- 환경 추가가 간단 (ApplicationSet + Overlay)

### 단점
- DEV 클러스터를 ArgoCD에 별도 등록 필요
- 클러스터 간 네트워크 설정 필요

### 비용 비교

| 방식 | 클러스터 수 | 월 예상 비용 |
|------|-------------|--------------|
| 단일 클러스터 | 1 | ~$150 |
| **PRD/DEV 분리 (채택)** | 2 | ~$300 |
| 완전 분리 | 4+ | ~$600+ |

## 관련 파일

- `argocd/appsets/cms.yaml` - CMS ApplicationSet
- `argocd/appsets/ui.yaml` - UI ApplicationSet
- `scripts/09-register-dev-cluster.sh` - DEV 클러스터 등록 스크립트

## 참고

- ArgoCD Multi-Cluster: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters
- ApplicationSet Generators: https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/
