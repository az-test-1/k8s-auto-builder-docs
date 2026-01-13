# ADR-002: 다중 환경 배포를 위한 ApplicationSet 사용

## 상태
채택됨 (Accepted)

## 날짜
2026-01-12

## 컨텍스트

현재 환경 구성:
- **PRD 클러스터**: prd namespace (운영)
- **DEV 클러스터**: dev, stage, sandbox, test 등 다중 namespace (개발/테스트)

기존 방식은 환경별로 개별 ArgoCD Application 파일을 생성:
```
argocd/apps/
├── cms.yaml        # PRD
├── cms-dev.yaml    # DEV
├── cms-stage.yaml  # Stage (추가 필요)
├── cms-test.yaml   # Test (추가 필요)
└── ...
```

문제점:
- 환경 × 앱 수 만큼 파일 증가 (환경 5개 × 앱 2개 = 10개 파일)
- 동일한 설정의 반복 (boilerplate)
- 환경 추가 시 여러 파일 생성 필요

## 고려한 대안

### 대안 1: 개별 Application 파일 유지
- 장점: 단순, 명확
- 단점: 파일 수 증가, 중복 코드

### 대안 2: ApplicationSet 사용 (채택)
- 장점: 템플릿 기반 자동 생성, 환경 추가 용이
- 단점: 약간의 학습 곡선

### 대안 3: Helm + values 파일
- 장점: 유연한 템플릿
- 단점: 복잡도 증가, Kustomize와 혼용 시 혼란

## 결정

**ArgoCD ApplicationSet의 List Generator를 사용한다.**

### 구조
```
argocd/appsets/
├── cms.yaml    # 모든 환경의 CMS Application 생성
└── ui.yaml     # 모든 환경의 UI Application 생성
```

### ApplicationSet 템플릿
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: ab-cms
spec:
  generators:
    - list:
        elements:
          - env: prd
          - env: dev
          - env: stage    # 필요시 추가
          - env: test     # 필요시 추가
  template:
    metadata:
      name: 'ab-cms-{{env}}'
    spec:
      source:
        path: 'k8s/overlays/{{env}}/cms'
      destination:
        namespace: '{{env}}'
```

### 환경 추가 절차
1. `argocd/appsets/cms.yaml`, `ui.yaml`에 환경 추가:
   ```yaml
   elements:
     - env: newenv
   ```
2. `k8s/overlays/newenv/` 폴더 생성 (기존 환경 복사 후 수정)
3. 커밋 & 푸시 → ArgoCD 자동 감지 및 배포

## 결과

### 장점
- 파일 수 대폭 감소 (10개 → 2개)
- 환경 추가 시 2줄만 추가
- 일관된 Application 설정 보장
- 환경별 차이는 Kustomize overlay에서 관리

### 단점
- ApplicationSet 개념 학습 필요
- 환경별로 완전히 다른 설정이 필요한 경우 별도 Application 필요

## 클러스터 구성

```
┌─────────────────────────────┐
│   aks-cloudtr-prd (운영)     │
│   └── namespace: prd        │
└─────────────────────────────┘

┌─────────────────────────────┐
│   aks-cloudtr-dev (개발)     │
│   ├── namespace: dev        │
│   ├── namespace: stage      │
│   ├── namespace: sandbox    │
│   └── namespace: test       │
└─────────────────────────────┘
```

- PRD 클러스터: 운영 환경만 (완전 격리)
- DEV 클러스터: 모든 비운영 환경 (비용 효율)

## 참고
- ArgoCD ApplicationSet 문서: https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/
- List Generator 문서: https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-List/
