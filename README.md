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
├── README.md                    # 이 문서
├── prd/                         # 제품 요구사항 문서 (PRD)
│   ├── PRD-000-MASTER.md       # 마스터 PRD
│   ├── PRD-010-Core-Migration.md
│   ├── PRD-020-AI-LLM.md
│   ├── PRD-030-Admin-Dashboard.md
│   ├── PRD-040-Platform-Infra.md
│   └── SCR-*.md                # 화면별 상세 PRD
├── adr/                         # 아키텍처 결정 기록 (ADR)
│   ├── README.md
│   ├── 001-006                 # 일반 ADR
│   └── M001-M014               # 마이그레이션 ADR
├── features/                    # 기능 개발 문서
│   └── _templates/             # 문서 템플릿
└── [카테고리]/*.md              # 분석/설계/학습/참고 문서
```

---

## 주요 문서

### PRD (Product Requirements Document)

제품 요구사항 및 화면 설계 문서입니다.

| 문서 | 설명 |
|------|------|
| [PRD-000-MASTER](./prd/PRD-000-MASTER.md) | 마스터 PRD - 제품 전체 개요 |
| [PRD-010-Core-Migration](./prd/PRD-010-Core-Migration.md) | 핵심 마이그레이션 기능 |
| [PRD-020-AI-LLM](./prd/PRD-020-AI-LLM.md) | AI/LLM 에러 수정 기능 |
| [PRD-030-Admin-Dashboard](./prd/PRD-030-Admin-Dashboard.md) | 관리자 대시보드 |
| [PRD-040-Platform-Infra](./prd/PRD-040-Platform-Infra.md) | 플랫폼 인프라 |

#### 화면별 PRD (SCR)

| 화면 ID | 화면명 | 도메인 |
|---------|--------|--------|
| [SCR-001](./prd/SCR-001-Login.md) | 로그인 | 인증 |
| [SCR-002](./prd/SCR-002-Layout.md) | 레이아웃 | 공통 |
| [SCR-010](./prd/SCR-010-Dashboard.md) | 대시보드 | 메인 |
| [SCR-020](./prd/SCR-020-ProjectList.md) | 프로젝트 목록 | 프로젝트 |
| [SCR-021](./prd/SCR-021-ProjectCreate.md) | 프로젝트 생성 | 프로젝트 |
| [SCR-022](./prd/SCR-022-ProjectDetail.md) | 프로젝트 상세 | 프로젝트 |
| [SCR-030](./prd/SCR-030-ExecutionRun.md) | 실행 | 실행 |
| [SCR-031](./prd/SCR-031-ExecutionMonitor.md) | 실행 모니터링 | 실행 |
| [SCR-032](./prd/SCR-032-ExecutionHistory.md) | 실행 이력 | 실행 |
| [SCR-033](./prd/SCR-033-ExecutionResult.md) | 실행 결과 | 실행 |
| [SCR-040](./prd/SCR-040-UserManagement.md) | 사용자 관리 | 관리자 |
| [SCR-041](./prd/SCR-041-GroupManagement.md) | 그룹 관리 | 관리자 |
| [SCR-042](./prd/SCR-042-SystemSettings.md) | 시스템 설정 | 관리자 |
| [SCR-050](./prd/SCR-050-PromptManagement.md) | 프롬프트 관리 | AI/LLM |
| [SCR-051](./prd/SCR-051-FewshotManagement.md) | Few-shot 관리 | AI/LLM |

---

### ADR (Architecture Decision Records)

아키텍처 결정 기록입니다. 자세한 내용은 [adr/README.md](./adr/README.md)를 참조하세요.

| 시리즈 | 범위 | 문서 수 |
|--------|------|:-------:|
| [001-006](./adr/README.md#adr-목록) | 일반 시스템 결정 | 7 |
| [M001-M014](./adr/README.md#m-시리즈-docker-to-kubernetes-마이그레이션) | Docker to K8s 마이그레이션 | 14 |

---

### 분석 문서

현황 분석 및 진단 문서입니다.

| 문서 | 설명 |
|------|------|
| [12-factor-cloud-native-checklist-v2](./%5B분석%5D%2012-factor-cloud-native-checklist-v2-P0완료.md) | 12-Factor 클라우드 네이티브 체크리스트 |
| [CICD-Kustomize-역할분담](./%5B분석%5D%20CICD-Kustomize-역할분담.md) | CI/CD와 Kustomize 역할 분담 |
| [CICD-Kustomize-진단결과](./%5B분석%5D%20CICD-Kustomize-진단결과.md) | CI/CD 및 Kustomize 진단 |
| [스토리지-구성-비교분석](./%5B분석%5D%20스토리지-구성-비교분석.md) | Azure 스토리지 옵션 비교 |
| [유즈케이스-Java8to17-실행흐름](./%5B분석%5D%20유즈케이스-Java8to17-실행흐름.md) | Java 8→17 마이그레이션 플로우 |
| [환경변수-설정현황](./%5B분석%5D%20환경변수-설정현황.md) | 환경변수 설정 현황 |
| [환경별-설정-현황](./%5B분석%5D%20환경별-설정-현황.md) | DEV/PRD 환경별 설정 |

---

### 설계 문서

아키텍처 및 시스템 설계 문서입니다.

| 문서 | 설명 |
|------|------|
| [aks-architecture](./%5B설계%5D%20aks-architecture.md) | AKS 아키텍처 설계 |
| [대규모-스토리지-아키텍처](./%5B설계%5D%20대규모-스토리지-아키텍처.md) | 대규모 스토리지 설계 |

---

### 학습 문서

Kubernetes 및 Azure 관련 학습 자료입니다.

| 문서 | 설명 |
|------|------|
| [HPA](./%5B학습%5D%20HPA.md) | Horizontal Pod Autoscaler |
| [Ingress](./%5B학습%5D%20Ingress.md) | Kubernetes Ingress |
| [이미지 관련](./%5B학습%5D%20이미지%20관련.md) | 컨테이너 이미지 관리 |
| [P0-Cloud-Native-개선작업-상세가이드](./%5B학습%5D%20P0-Cloud-Native-개선작업-상세가이드.md) | 클라우드 네이티브 개선 가이드 |

---

### 참고 문서

| 문서 | 설명 |
|------|------|
| [PDB-PodDisruptionBudget](./%5B참고%5D%20PDB-PodDisruptionBudget.md) | Pod Disruption Budget 참고 |
| [Azure 리소스 스펙 변경 시 영향도 분석](./Azure%20리소스%20스펙%20변경%20시%20영향도%20분석.md) | Azure 리소스 변경 영향 분석 |

---

### 아카이브

더 이상 활성화되지 않은 문서입니다.

| 문서 | 상태 |
|------|------|
| [12-factor-cloud-native-checklist](./%5B아카이브%5D%2012-factor-cloud-native-checklist.md) | v2로 대체됨 |
| [신규환경-배포-체크리스트](./%5B아카이브%5D%20신규환경-배포-체크리스트.md) | 완료됨 |

---

## 관련 저장소

| 저장소 | 설명 |
|--------|------|
| [k8s-auto-builder-cms](https://github.com/az-test-1/k8s-auto-builder-cms) | 백엔드 API (FastAPI) |
| [k8s-auto-builder-ui](https://github.com/az-test-1/k8s-auto-builder-ui) | 프론트엔드 (Next.js) |
| [k8s-auto-builder-infra](https://github.com/az-test-1/k8s-auto-builder-infra) | 인프라 (Kubernetes, Azure) |

---

## 기여 가이드

### 문서 작성 규칙

1. **파일명**: 한글 사용 가능, 공백 대신 `-` 사용
2. **카테고리 접두사**: `[분석]`, `[설계]`, `[학습]`, `[참고]`, `[아카이브]`
3. **ADR**: `adr/` 폴더에 번호-제목 형식
4. **PRD**: `prd/` 폴더에 PRD-XXX 또는 SCR-XXX 형식

### 템플릿

- [ADR 템플릿](./features/_templates/ADR.md)
- [PRD 템플릿](./features/_templates/PRD.md)
- [CHANGELOG 템플릿](./features/_templates/CHANGELOG.md)

---

## 변경 이력

| 날짜 | 변경 내용 |
|------|----------|
| 2026-01-13 | 초기 문서 구조 생성 |
| 2026-01-13 | M 시리즈 ADR 추가 (Docker to K8s 마이그레이션) |
| 2026-01-13 | 화면별 PRD (SCR-001 ~ SCR-051) 추가 |

---

*최종 업데이트: 2026-01-13*
