# Architecture Decision Records (ADR)

> 최종 업데이트: 2026-01-13

시스템 레벨의 아키텍처 결정 기록입니다.

## ADR 목록

| # | 제목 | 상태 | 범위 |
|---|------|------|------|
| [ADR-001](./ADR-001-namespace-management.md) | 네임스페이스 관리 단일화 | 채택됨 | 인프라 |
| [ADR-002](./ADR-002-applicationset.md) | 다중 환경 배포용 ApplicationSet | 채택됨 | 인프라 |
| [ADR-003](./ADR-003-k8s-naming.md) | K8s 리소스 명명 규칙 | 채택됨 | 시스템 |
| [ADR-004](./ADR-004-image-terminology.md) | 이미지 용어 정의 및 분류 | 제안됨 | 시스템 |
| [ADR-005](./ADR-005-multi-cluster.md) | 멀티 클러스터 배포 전략 | 채택됨 | 인프라 |
| [ADR-006](./ADR-006-scaling-strategy.md) | 동시 실행 스케일링 전략 | 분석완료 | 인프라 |
| [ADR-007](./ADR-007-docker-to-k8s-migration.md) | Docker to K8s 마이그레이션 (통합본) | 채택됨 | 시스템 |

---

## ADR-M 시리즈: Docker to Kubernetes 마이그레이션

> ADR-007의 개별 항목을 분리한 상세 문서입니다.

| # | 제목 | 상태 | 카테고리 |
|---|------|:----:|----------|
| [ADR-M001](./migrations/ADR-M001-container-orchestration.md) | 컨테이너 오케스트레이션 플랫폼 선택 | ✅ | Infrastructure |
| [ADR-M002](./migrations/ADR-M002-resource-groups.md) | 환경별 리소스 그룹 분리 | ✅ | Infrastructure |
| [ADR-M003](./migrations/ADR-M003-namespace-strategy.md) | Kubernetes 네임스페이스 전략 | ✅ | Kubernetes |
| [ADR-M004](./migrations/ADR-M004-database.md) | 데이터베이스 선택 | ✅ | Database |
| [ADR-M005](./migrations/ADR-M005-storage.md) | 스토리지 솔루션 | ✅ | Storage |
| [ADR-M006](./migrations/ADR-M006-secrets.md) | 시크릿 관리 방식 | ✅ | Security |
| [ADR-M007](./migrations/ADR-M007-ingress.md) | Ingress Controller 선택 | ✅ | Networking |
| [ADR-M008](./migrations/ADR-M008-container-registry.md) | 컨테이너 레지스트리 전략 | ✅ | Container |
| [ADR-M009](./migrations/ADR-M009-cicd.md) | CI/CD 파이프라인 및 배포 도구 | ✅ | CI/CD |
| [ADR-M010](./migrations/ADR-M010-build-jobs.md) | 빌드 Job 실행 방식 | ✅ | Build |
| [ADR-M011](./migrations/ADR-M011-high-availability.md) | 고가용성 전략 (HPA/PDB) | ✅ | High Availability |
| [ADR-M012](./migrations/ADR-M012-redis.md) | Redis 배포 방식 | ✅ | Cache |
| [ADR-M013](./migrations/ADR-M013-logging-monitoring.md) | 로깅 및 모니터링 | 🔄 | Observability |
| [ADR-M014](./migrations/ADR-M014-network-policy.md) | 네트워크 정책 | 🔄 | Security |

> ✅ 승인됨 | 🔄 진행 중/보류

---

## 상태 정의

| 상태 | 설명 |
|------|------|
| 제안됨 | 검토 중 |
| 채택됨 | 결정 확정 |
| 분석완료 | 현황 분석 완료 |
| 폐기됨 | 더 이상 유효하지 않음 |
| 대체됨 | 새로운 ADR로 대체됨 |

---

## ADR 작성 가이드

### 템플릿

```markdown
# ADR-XXX: 제목

## 상태
제안됨/채택됨/폐기됨/대체됨

## 날짜
YYYY-MM-DD

## 컨텍스트
결정이 필요한 배경과 현재 상황

## 고려한 대안
검토한 옵션들과 각각의 장단점

## 결정
최종 결정 내용과 구체적인 구현 방법

## 결과
결정으로 인한 장점, 단점, 영향

## 참고
관련 문서, 링크
```
