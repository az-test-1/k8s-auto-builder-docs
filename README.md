# Auto-Builder Documentation

> Auto-Builder 시스템의 기술 문서 저장소

[![GitHub](https://img.shields.io/badge/GitHub-az--test--1-blue)](https://github.com/az-test-1/k8s-auto-builder-docs)

---

## 개요

Auto-Builder는 AI 기반의 자동화된 Java 마이그레이션 및 빌드 시스템입니다.
레거시 Java 프로젝트(Java 8, Spring Boot 2)를 최신 버전(Java 17, Spring Boot 3)으로 자동 변환하고,
LLM을 활용하여 빌드 에러를 자동으로 수정합니다.

---

## 디렉토리 구조

```
k8s-auto-builder-docs/
├── README.md
├── 01-requirements/          # 요구사항
│   ├── prd/                  # PRD (제품 요구사항)
│   ├── screens/              # SCR (화면 명세)
│   ├── features/             # FTR (기능 개발)
│   ├── analysis/             # ANA (분석)
│   └── design/               # DES (설계)
├── 02-decisions/             # ADR (아키텍처 결정)
│   └── migrations/           # ADR-M (마이그레이션)
└── 03-guides/                # GUD (가이드)
    └── archive/              # ARC (아카이브)
```

---

## 01-requirements (요구사항)

### PRD (제품 요구사항)

| 문서 | 설명 |
|------|------|
| [PRD-000-MASTER](./01-requirements/prd/PRD-000-MASTER.md) | 마스터 PRD - 제품 전체 개요 |
| [PRD-010-Core-Migration](./01-requirements/prd/PRD-010-Core-Migration.md) | 핵심 마이그레이션 기능 |
| [PRD-020-AI-LLM](./01-requirements/prd/PRD-020-AI-LLM.md) | AI/LLM 에러 수정 기능 |
| [PRD-030-Admin-Dashboard](./01-requirements/prd/PRD-030-Admin-Dashboard.md) | 관리자 대시보드 |
| [PRD-040-Platform-Infra](./01-requirements/prd/PRD-040-Platform-Infra.md) | 플랫폼 인프라 |

### SCR (화면 명세)

| 화면 ID | 화면명 | 도메인 |
|---------|--------|--------|
| [SCR-001](./01-requirements/screens/SCR-001-Login.md) | 로그인 | 인증 |
| [SCR-002](./01-requirements/screens/SCR-002-Layout.md) | 레이아웃 | 공통 |
| [SCR-010](./01-requirements/screens/SCR-010-Dashboard.md) | 대시보드 | 메인 |
| [SCR-020](./01-requirements/screens/SCR-020-ProjectList.md) | 프로젝트 목록 | 프로젝트 |
| [SCR-021](./01-requirements/screens/SCR-021-ProjectCreate.md) | 프로젝트 생성 | 프로젝트 |
| [SCR-022](./01-requirements/screens/SCR-022-ProjectDetail.md) | 프로젝트 상세 | 프로젝트 |
| [SCR-030](./01-requirements/screens/SCR-030-ExecutionRun.md) | 실행 | 실행 |
| [SCR-031](./01-requirements/screens/SCR-031-ExecutionMonitor.md) | 실행 모니터링 | 실행 |
| [SCR-032](./01-requirements/screens/SCR-032-ExecutionHistory.md) | 실행 이력 | 실행 |
| [SCR-033](./01-requirements/screens/SCR-033-ExecutionResult.md) | 실행 결과 | 실행 |
| [SCR-040](./01-requirements/screens/SCR-040-UserManagement.md) | 사용자 관리 | 관리자 |
| [SCR-041](./01-requirements/screens/SCR-041-GroupManagement.md) | 그룹 관리 | 관리자 |
| [SCR-042](./01-requirements/screens/SCR-042-SystemSettings.md) | 시스템 설정 | 관리자 |
| [SCR-050](./01-requirements/screens/SCR-050-PromptManagement.md) | 프롬프트 관리 | AI/LLM |
| [SCR-051](./01-requirements/screens/SCR-051-FewshotManagement.md) | Few-shot 관리 | AI/LLM |

### FTR (기능 개발)

| 문서 | 설명 |
|------|------|
| [FTR-001-prompt-access-control](./01-requirements/features/FTR-001-prompt-access-control.md) | 프롬프트 접근 제어 |
| [FTR-002-pre-validation](./01-requirements/features/FTR-002-pre-validation.md) | 사전 검증 |

### ANA (분석)

| 문서 | 설명 |
|------|------|
| [ANA-001-cloud-native-checklist](./01-requirements/analysis/ANA-001-cloud-native-checklist.md) | 12-Factor 클라우드 네이티브 체크리스트 |
| [ANA-002-cicd-kustomize-roles](./01-requirements/analysis/ANA-002-cicd-kustomize-roles.md) | CI/CD와 Kustomize 역할 분담 |
| [ANA-003-cicd-kustomize-diagnosis](./01-requirements/analysis/ANA-003-cicd-kustomize-diagnosis.md) | CI/CD 및 Kustomize 진단 |
| [ANA-004-storage-comparison](./01-requirements/analysis/ANA-004-storage-comparison.md) | Azure 스토리지 옵션 비교 |
| [ANA-005-java-migration-flow](./01-requirements/analysis/ANA-005-java-migration-flow.md) | Java 8→17 마이그레이션 플로우 |
| [ANA-006-env-variables](./01-requirements/analysis/ANA-006-env-variables.md) | 환경변수 설정 현황 |
| [ANA-007-env-config](./01-requirements/analysis/ANA-007-env-config.md) | DEV/PRD 환경별 설정 |

### DES (설계)

| 문서 | 설명 |
|------|------|
| [DES-001-aks-architecture](./01-requirements/design/DES-001-aks-architecture.md) | AKS 아키텍처 설계 |
| [DES-002-storage-architecture](./01-requirements/design/DES-002-storage-architecture.md) | 대규모 스토리지 설계 |
| [DES-003-validation-pipeline](./01-requirements/design/DES-003-validation-pipeline.md) | 실행환경 검증 파이프라인 |

---

## 02-decisions (아키텍처 결정)

### ADR

| # | 제목 | 상태 |
|---|------|------|
| [ADR-001](./02-decisions/ADR-001-namespace-management.md) | 네임스페이스 관리 단일화 | 채택됨 |
| [ADR-002](./02-decisions/ADR-002-applicationset.md) | 다중 환경 배포용 ApplicationSet | 채택됨 |
| [ADR-003](./02-decisions/ADR-003-k8s-naming.md) | K8s 리소스 명명 규칙 | 채택됨 |
| [ADR-004](./02-decisions/ADR-004-image-terminology.md) | 이미지 용어 정의 및 분류 | 제안됨 |
| [ADR-005](./02-decisions/ADR-005-multi-cluster.md) | 멀티 클러스터 배포 전략 | 채택됨 |
| [ADR-006](./02-decisions/ADR-006-scaling-strategy.md) | 동시 실행 스케일링 전략 | 분석완료 |
| [ADR-007](./02-decisions/ADR-007-docker-to-k8s-migration.md) | Docker to K8s 마이그레이션 (통합본) | 채택됨 |

### ADR-M (마이그레이션)

| # | 제목 | 상태 | 카테고리 |
|---|------|:----:|----------|
| [ADR-M001](./02-decisions/migrations/ADR-M001-container-orchestration.md) | 컨테이너 오케스트레이션 플랫폼 선택 | ✅ | Infrastructure |
| [ADR-M002](./02-decisions/migrations/ADR-M002-resource-groups.md) | 환경별 리소스 그룹 분리 | ✅ | Infrastructure |
| [ADR-M003](./02-decisions/migrations/ADR-M003-namespace-strategy.md) | Kubernetes 네임스페이스 전략 | ✅ | Kubernetes |
| [ADR-M004](./02-decisions/migrations/ADR-M004-database.md) | 데이터베이스 선택 | ✅ | Database |
| [ADR-M005](./02-decisions/migrations/ADR-M005-storage.md) | 스토리지 솔루션 | ✅ | Storage |
| [ADR-M006](./02-decisions/migrations/ADR-M006-secrets.md) | 시크릿 관리 방식 | ✅ | Security |
| [ADR-M007](./02-decisions/migrations/ADR-M007-ingress.md) | Ingress Controller 선택 | ✅ | Networking |
| [ADR-M008](./02-decisions/migrations/ADR-M008-container-registry.md) | 컨테이너 레지스트리 전략 | ✅ | Container |
| [ADR-M009](./02-decisions/migrations/ADR-M009-cicd.md) | CI/CD 파이프라인 및 배포 도구 | ✅ | CI/CD |
| [ADR-M010](./02-decisions/migrations/ADR-M010-build-jobs.md) | 빌드 Job 실행 방식 | ✅ | Build |
| [ADR-M011](./02-decisions/migrations/ADR-M011-high-availability.md) | 고가용성 전략 (HPA/PDB) | ✅ | High Availability |
| [ADR-M012](./02-decisions/migrations/ADR-M012-redis.md) | Redis 배포 방식 | ✅ | Cache |
| [ADR-M013](./02-decisions/migrations/ADR-M013-logging-monitoring.md) | 로깅 및 모니터링 | 🔄 | Observability |
| [ADR-M014](./02-decisions/migrations/ADR-M014-network-policy.md) | 네트워크 정책 | 🔄 | Security |

> ✅ 승인됨 | 🔄 진행 중/보류

---

## 03-guides (가이드)

### GUD (가이드)

| 문서 | 설명 |
|------|------|
| [GUD-001-hpa](./03-guides/GUD-001-hpa.md) | Horizontal Pod Autoscaler |
| [GUD-002-ingress](./03-guides/GUD-002-ingress.md) | Kubernetes Ingress |
| [GUD-003-container-images](./03-guides/GUD-003-container-images.md) | 컨테이너 이미지 관리 |
| [GUD-004-cloud-native-improvements](./03-guides/GUD-004-cloud-native-improvements.md) | 클라우드 네이티브 개선 가이드 |
| [GUD-005-pdb](./03-guides/GUD-005-pdb.md) | Pod Disruption Budget |

### ARC (아카이브)

| 문서 | 상태 |
|------|------|
| [ARC-001-cloud-native-checklist-old](./03-guides/archive/ARC-001-cloud-native-checklist-old.md) | v2로 대체됨 |
| [ARC-002-deployment-checklist](./03-guides/archive/ARC-002-deployment-checklist.md) | 완료됨 |

---

## 관련 저장소

| 저장소 | 설명 |
|--------|------|
| [k8s-auto-builder-cms](https://github.com/az-test-1/k8s-auto-builder-cms) | 백엔드 API (FastAPI) |
| [k8s-auto-builder-ui](https://github.com/az-test-1/k8s-auto-builder-ui) | 프론트엔드 (Next.js) |
| [k8s-auto-builder-infra](https://github.com/az-test-1/k8s-auto-builder-infra) | 인프라 (Kubernetes, Azure) |

---

## 문서 코드 규칙

| 접두사 | 의미 | 위치 |
|--------|------|------|
| `PRD-` | Product Requirements | `01-requirements/prd/` |
| `SCR-` | Screen Specification | `01-requirements/screens/` |
| `FTR-` | Feature | `01-requirements/features/` |
| `ANA-` | Analysis | `01-requirements/analysis/` |
| `DES-` | Design | `01-requirements/design/` |
| `ADR-` | Architecture Decision | `02-decisions/` |
| `ADR-M` | Migration Decision | `02-decisions/migrations/` |
| `GUD-` | Guide | `03-guides/` |
| `ARC-` | Archive | `03-guides/archive/` |

---

## 변경 이력

| 날짜 | 변경 내용 |
|------|----------|
| 2026-01-13 | 문서 구조 개편 (Arc42 간략화 적용) |
| 2026-01-13 | 실행환경 검증 파이프라인 설계 추가 |
| 2026-01-13 | M 시리즈 ADR 추가 |

---

*최종 업데이트: 2026-01-13*
