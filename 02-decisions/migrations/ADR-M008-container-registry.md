# ADR-M008: 컨테이너 레지스트리 전략

> **Status**: 승인됨
> **Date**: 2025-12
> **Category**: Migration - Container

---

## 상태
✅ **승인됨** (2025-12)

## 컨텍스트
컨테이너 이미지 저장소 선택 및 DEV/PRD 환경 공유 여부 결정 필요

## 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. Docker Hub** | 무료, 간편 | Rate Limit, 보안 우려 |
| **B. GitHub Container Registry** | GitHub 통합 | Azure 통합 약함 |
| **C. Azure Container Registry (환경별)** | 완전 격리 | 비용 증가, 이미지 중복 |
| **D. Azure Container Registry (공용)** | 비용 절감, 이미지 공유 | 권한 관리 주의 |

## 결정
**옵션 D: Azure Container Registry 공용 (단일 ACR)**

## 근거
1. **비용 효율**: ACR 하나로 DEV/PRD 모두 사용
2. **이미지 공유**: 동일 이미지를 DEV에서 테스트 후 PRD에 배포
3. **Azure 통합**: AKS와 원활한 연동 (Managed Identity)
4. **빌드 이미지 공유**: 빌드 Job용 기본 이미지 (Maven, Gradle 등) 공유

## ACR 설정
```
Name: acraz01cloudtr
SKU: Basic
Location: Korea Central
Resource Group: rg-cloudtr-prd (PRD에서 관리)
```

## AKS 연결
```bash
# 각 AKS에 ACR 연결
az aks update -n aks-cloudtr-dev -g rg-cloudtr-dev --attach-acr acraz01cloudtr
az aks update -n aks-cloudtr-prd -g rg-cloudtr-prd --attach-acr acraz01cloudtr
```

## 이미지 태그 전략
```
# 애플리케이션 이미지
acraz01cloudtr.azurecr.io/auto-builder-cms:dev-abc123
acraz01cloudtr.azurecr.io/auto-builder-cms:prd-def456
acraz01cloudtr.azurecr.io/auto-builder-ui:dev-abc123
acraz01cloudtr.azurecr.io/auto-builder-ui:prd-def456

# 빌드 이미지 (버전 고정)
acraz01cloudtr.azurecr.io/java17-maven3.9.6:latest
acraz01cloudtr.azurecr.io/java17-gradle8:latest
acraz01cloudtr.azurecr.io/python:3.12-slim-bookworm
```

## 관련 파일
```
azure/scripts/02-acr.sh
```

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
