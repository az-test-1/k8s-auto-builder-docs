# ADR-M002: 환경별 리소스 그룹 분리

> **Status**: 승인됨
> **Date**: 2026-01
> **Category**: Migration - Infrastructure

---

## 상태
✅ **승인됨** (2026-01)

## 컨텍스트
초기에는 단일 리소스 그룹(`rg-cloudtr-aks`)에 DEV/PRD 클러스터를 모두 배치했으나, 관리 및 권한 분리 필요성 대두

## 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. 단일 리소스 그룹** | 관리 단순 | 권한 분리 어려움, 비용 추적 어려움 |
| **B. 환경별 리소스 그룹 분리** | 권한/비용 분리, 명확한 경계 | 관리 대상 증가 |
| **C. 서비스별 리소스 그룹** | 세밀한 제어 | 과도한 복잡성 |

## 결정
**옵션 B: 환경별 리소스 그룹 분리**

```
rg-cloudtr-dev  → DEV 환경 모든 리소스
rg-cloudtr-prd  → PRD 환경 모든 리소스
```

## 근거
1. **권한 분리**: DEV/PRD 접근 권한 별도 관리 가능
2. **비용 추적**: 환경별 비용 명확히 구분
3. **리소스 격리**: 실수로 인한 크로스 환경 영향 방지
4. **정책 적용**: 환경별 다른 Azure Policy 적용 가능

## 결과
- DEV: `rg-cloudtr-dev` (모든 DEV 리소스)
- PRD: `rg-cloudtr-prd` (모든 PRD 리소스)
- ACR만 PRD 리소스 그룹에서 공용으로 사용

## 변경된 파일
```
azure/scripts/00-env.sh
.github/workflows/ci-cd.yaml
kubernetes/overlays/*/kustomization.yaml
```

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
