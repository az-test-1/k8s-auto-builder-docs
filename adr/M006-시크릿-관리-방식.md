# ADR-M006: 시크릿 관리 방식

> **Status**: 승인됨
> **Date**: 2025-12
> **Category**: Migration - Security

---

## 상태
✅ **승인됨** (2025-12)

## 컨텍스트
DB 비밀번호, API 키 등 민감 정보를 안전하게 관리하고 Pod에 주입하는 방법 결정 필요

## 고려한 옵션

| 옵션 | 보안 | 운영 편의 | 감사 |
|------|:----:|:--------:|:----:|
| **A. K8s Secret (직접 생성)** | 낮음 | 높음 | 낮음 |
| **B. Sealed Secrets** | 중간 | 중간 | 중간 |
| **C. Azure Key Vault + CSI Driver** | 높음 | 중간 | 높음 |
| **D. HashiCorp Vault** | 높음 | 낮음 | 높음 |
| **E. External Secrets Operator** | 높음 | 높음 | 높음 |

## 결정
**옵션 C: Azure Key Vault + Secrets Store CSI Driver**
(CI/CD에서 Key Vault → K8s Secret 변환 방식 병행)

## 근거
1. **Azure 네이티브**: 추가 인프라 없이 Azure Key Vault 활용
2. **중앙 집중 관리**: 모든 시크릿을 Key Vault에서 통합 관리
3. **감사 로그**: Azure Key Vault의 접근 로그 자동 기록
4. **RBAC 통합**: Azure RBAC으로 접근 권한 관리
5. **CI/CD 연동**: GitHub Actions에서 Key Vault 참조하여 K8s Secret 생성

## 구현 방식
```
[Key Vault] → [GitHub Actions] → [K8s Secret] → [Pod]
     │              │
     │              └── az keyvault secret show
     │                  kubectl create secret
     │
     └── CSI Driver 방식 (대안)
```

## Key Vault 시크릿 목록
```
DATABASE-URL
JWT-SECRET-KEY
INIT-PASSWORD
REDIS-PASSWORD
NEXTAUTH-SECRET
SERVICE-ACR-PASSWORD
OPENAI-KEY-1 ~ 6
CLAUDE-KEY
```

## 관련 파일
```
azure/scripts/02a-keyvault.sh
.github/workflows/ci-cd.yaml (secrets 섹션)
```

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
