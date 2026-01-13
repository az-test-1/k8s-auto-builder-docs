# ADR-001: Namespace 관리 단일화

## 상태
채택됨 (Accepted)

## 날짜
2026-01-12

## 컨텍스트

GitOps 환경에서 Kubernetes namespace를 정의하는 위치가 여러 곳에 분산되어 있었다:

1. `base/cms/namespace.yaml` - Namespace 리소스 정의
2. `overlays/{env}/cms/kustomization.yaml` - namespace 필드
3. `argocd/apps/*.yaml` - destination.namespace

이로 인해:
- 동일한 정보가 중복 관리됨
- 환경 추가 시 여러 파일을 수정해야 함
- 불일치 발생 가능성 존재

## 결정

**ArgoCD Application의 `destination.namespace`를 단일 소스로 사용한다.**

### 제거한 것
- `base/cms/namespace.yaml` 파일 삭제
- `overlays/{env}/*/kustomization.yaml`의 `namespace:` 필드 삭제
- namespace patch 삭제

### 유지한 것
- ArgoCD Application의 `destination.namespace` 설정
- `syncOptions: CreateNamespace=true`로 자동 생성

## 결과

### 장점
- 단일 관리 포인트 (Single Source of Truth)
- 환경 추가 시 ArgoCD Application만 수정
- 설정 불일치 방지
- Kustomize overlay가 더 간결해짐

### 단점
- ArgoCD 없이 `kubectl apply -k`로 직접 배포 시 namespace 수동 생성 필요
  - 대응: 개발/운영 모두 ArgoCD를 통해서만 배포하므로 문제없음

## 참고
- ArgoCD 공식 문서: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/#create-namespace
