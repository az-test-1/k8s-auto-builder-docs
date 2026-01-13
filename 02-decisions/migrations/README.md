# Migration ADRs (Architecture Decision Records)

> Docker Compose → Kubernetes 마이그레이션 결정 기록

## 개요

이 폴더에는 Auto-Builder 시스템을 Docker Compose 기반에서 Azure Kubernetes Service(AKS)로 마이그레이션하면서 내린 아키텍처 결정들을 기록합니다.

## 문서 목록

| 문서 | 제목 | 카테고리 |
|------|------|----------|
| [ADR-M001](./ADR-M001-container-orchestration.md) | 컨테이너 오케스트레이션 플랫폼 선택 | Infrastructure |
| [ADR-M002](./ADR-M002-resource-groups.md) | Azure 리소스 그룹 구조 | Infrastructure |
| [ADR-M003](./ADR-M003-namespace-strategy.md) | Kubernetes 네임스페이스 전략 | Infrastructure |
| [ADR-M004](./ADR-M004-database.md) | 데이터베이스 구성 | Data |
| [ADR-M005](./ADR-M005-storage.md) | 스토리지 구성 | Data |
| [ADR-M006](./ADR-M006-secrets.md) | 시크릿 관리 | Security |
| [ADR-M007](./ADR-M007-ingress.md) | Ingress 구성 | Network |
| [ADR-M008](./ADR-M008-container-registry.md) | 컨테이너 레지스트리 | Infrastructure |
| [ADR-M009](./ADR-M009-cicd.md) | CI/CD 파이프라인 | DevOps |
| [ADR-M010](./ADR-M010-build-jobs.md) | 빌드 Job 실행 방식 | Build |
| [ADR-M011](./ADR-M011-high-availability.md) | 고가용성 구성 | Infrastructure |
| [ADR-M012](./ADR-M012-redis.md) | Redis 구성 | Data |
| [ADR-M013](./ADR-M013-logging-monitoring.md) | 로깅 및 모니터링 | Observability |
| [ADR-M014](./ADR-M014-network-policy.md) | 네트워크 정책 | Security |

## 문서 구조

각 ADR 문서는 다음 구조를 따릅니다:

```
# ADR-MXXX: 제목
- 상태 (승인됨/검토중/대체됨)
- 컨텍스트 (문제 상황)
- 고려한 옵션 (비교 테이블)
- 결정 (선택한 옵션)
- 근거 (선택 이유)
- 결과 (구현 결과)
- 관련 파일
```

## 카테고리별 분류

### Infrastructure
- M001: 컨테이너 오케스트레이션 → **AKS 선택**
- M002: 리소스 그룹 → **DEV/PRD 분리**
- M003: 네임스페이스 → **환경별 분리 (dev, prd)**
- M008: 컨테이너 레지스트리 → **ACR 공용 사용**
- M011: 고가용성 → **HPA + PDB**

### Data
- M004: 데이터베이스 → **Azure PostgreSQL Flexible Server**
- M005: 스토리지 → **Azure Files Premium**
- M012: Redis → **StatefulSet + PVC**

### Security
- M006: 시크릿 관리 → **Azure Key Vault + K8s Secret**
- M014: 네트워크 정책 → **기본 허용, 점진적 강화**

### Network
- M007: Ingress → **NGINX Ingress Controller**

### DevOps
- M009: CI/CD → **GitHub Actions + Kustomize**

### Build
- M010: 빌드 Job → **Kubernetes Jobs (DinD 대체)**

### Observability
- M013: 로깅/모니터링 → **Azure Monitor + Log Analytics**

---

## 관련 문서

- [02-decisions/README.md](../README.md) - 전체 ADR 목록
- [DES-001-aks-architecture](../../01-requirements/design/DES-001-aks-architecture.md) - AKS 아키텍처 설계
