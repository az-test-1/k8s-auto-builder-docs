# PRD-040: Platform & Infrastructure

> **Status**: Draft
> **Version**: 1.0.0
> **Last Updated**: 2026-01-13
> **Owner**: Platform Team
> **Parent**: [PRD-000-MASTER](./PRD-000-MASTER.md)

---

## 1. 개요

### 1.1 목적
Auto-Builder의 플랫폼 아키텍처 및 인프라 요구사항을 정의합니다.

### 1.2 범위
- 시스템 아키텍처
- 기술 스택
- 인프라 구성
- 비기능 요구사항 (성능, 확장성, 가용성, 보안)
- 모니터링 및 운영

---

## 2. 시스템 아키텍처

### 2.1 전체 구성도

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Client Layer                              │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Auto-Builder UI                            │  │
│  │                   (Next.js 15, React 19)                      │  │
│  └───────────────────────────────────────────────────────────────┘  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ HTTPS
┌────────────────────────────────▼────────────────────────────────────┐
│                         Kubernetes (AKS)                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      NGINX Ingress                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│            │                    │                    │              │
│            ▼                    ▼                    ▼              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │  cms-web-ui  │    │ cms-web-api  │    │ cms-web-mcp  │          │
│  │  (Next.js)   │    │  (FastAPI)   │    │  (MCP Server)│          │
│  └──────────────┘    └──────────────┘    └──────────────┘          │
│                              │                                      │
│                              ▼                                      │
│                    ┌──────────────────┐                            │
│                    │  Celery Worker   │                            │
│                    │ (비동기 작업 실행) │                            │
│                    └────────┬─────────┘                            │
│                             │                                      │
│            ┌────────────────┼────────────────┐                     │
│            ▼                ▼                ▼                     │
│    ┌────────────┐   ┌────────────┐   ┌─────────────────┐          │
│    │   Redis    │   │ K8s Jobs   │   │ Azure Files PVC │          │
│    │(Queue/Cache)│   │(빌드 컨테이너)│   │ (프로젝트 데이터)│          │
│    └────────────┘   └────────────┘   └─────────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                        External Services                            │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │  PostgreSQL  │    │     ACR      │    │  OpenAI/     │          │
│  │   (Azure)    │    │(이미지 레지스트리)│    │  Claude API  │          │
│  └──────────────┘    └──────────────┘    └──────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 컴포넌트 설명

| 컴포넌트 | 역할 | 기술 |
|----------|------|------|
| cms-web-ui | 웹 프론트엔드 | Next.js 15 |
| cms-web-api | REST API 서버 | FastAPI |
| cms-web-mcp | MCP 프로토콜 서버 | Python |
| Celery Worker | 비동기 작업 처리 | Celery 5.x |
| Redis | 메시지 큐, 캐시 | Redis 7.x |
| K8s Jobs | 빌드 작업 실행 | Kubernetes |

---

## 3. 기술 스택

### 3.1 Backend

| 구성요소 | 기술 | 버전 |
|----------|------|------|
| Framework | FastAPI | 0.100+ |
| Language | Python | 3.12 |
| Task Queue | Celery | 5.x |
| Message Broker | Redis | 7.x |
| Database | PostgreSQL | 16 |
| ORM | SQLAlchemy | 2.x |

### 3.2 Frontend

| 구성요소 | 기술 | 버전 |
|----------|------|------|
| Framework | Next.js | 15.x |
| Library | React | 19 |
| Styling | SCSS | - |
| Visualization | ReactFlow, Chart.js | - |
| Code Editor | CodeMirror | 6.x |

### 3.3 Infrastructure

| 구성요소 | 서비스 |
|----------|--------|
| Container Orchestration | Azure Kubernetes Service (AKS) |
| Container Registry | Azure Container Registry (ACR) |
| Database | Azure Database for PostgreSQL |
| Storage | Azure Files (Premium) |
| Secrets | Azure Key Vault |
| CI/CD | GitHub Actions |

---

## 4. 데이터 모델

### 4.1 Core Entities

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Group     │────<│   Project    │────<│     Run      │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ id           │     │ id           │     │ id           │
│ name         │     │ group_id     │     │ project_id   │
│ description  │     │ name         │     │ status       │
│ created_at   │     │ source_type  │     │ started_at   │
│              │     │ git_url      │     │ completed_at │
│              │     │ build_tool   │     │ error_message│
│              │     │ java_version │     │              │
│              │     │ spring_version│    │              │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                                  │
                                          ┌──────▼───────┐
                                          │   RunStep    │
                                          ├──────────────┤
                                          │ id           │
                                          │ run_id       │
                                          │ step_type    │
                                          │ status       │
                                          │ started_at   │
                                          │ completed_at │
                                          │ log_output   │
                                          └──────────────┘
```

---

## 5. 비기능 요구사항

### 5.1 성능 (Performance)

| 항목 | 요구사항 |
|------|----------|
| API 응답 시간 | < 500ms (95 percentile) |
| 동시 실행 프로젝트 | 최대 10개 |
| 단일 프로젝트 처리 시간 | < 30분 (중형 프로젝트 기준) |
| 로그 스트리밍 지연 | < 1초 |

### 5.2 확장성 (Scalability)

| 항목 | 요구사항 |
|------|----------|
| Horizontal Scaling | HPA 기반 Pod 자동 확장 (1-5 replicas) |
| 스토리지 | Azure Files Premium 100GiB (확장 가능) |
| 빌드 Job | 동시 최대 5개 K8s Job 실행 |

### 5.3 가용성 (Availability)

| 항목 | 요구사항 |
|------|----------|
| 서비스 가용성 | 99.5% 이상 |
| 계획된 다운타임 | 월 1회, 30분 이내 |
| 무중단 배포 | Rolling Update 적용 |
| PDB | 최소 1개 Pod 항상 가동 |

### 5.4 보안 (Security)

| 항목 | 요구사항 |
|------|----------|
| 인증 | NextAuth (JWT 기반) |
| 시크릿 관리 | Azure Key Vault |
| 네트워크 | AKS 내부 통신, Ingress 통한 외부 노출 |
| 이미지 보안 | ACR 프라이빗 레지스트리 사용 |
| DB 접근 | SSL 필수, AKS 서브넷에서만 접근 허용 |

### 5.5 모니터링 (Observability)

| 항목 | 요구사항 |
|------|----------|
| 로깅 | 구조화된 JSON 로그 |
| 메트릭 | Prometheus 형식 메트릭 노출 |
| 트레이싱 | 요청 ID 기반 추적 |
| 알림 | 실행 실패 시 알림 (Slack/Email) |

---

## 6. 인프라 구성

### 6.1 AKS 클러스터 스펙

| 항목 | PRD | DEV |
|------|-----|-----|
| Node VM Size | Standard_B2ms | Standard_B2ms |
| Node Count | 1-3 (Autoscale) | 1-2 (Autoscale) |
| Kubernetes Version | 1.32 | 1.32 |
| Network Plugin | Azure CNI | Azure CNI |

### 6.2 Azure 리소스

| 리소스 | 스펙 |
|--------|------|
| PostgreSQL | Standard_B1ms, 32GB |
| Storage Account | Standard_LRS, 10GB File Share |
| Key Vault | Standard |
| ACR | Basic |

### 6.3 리소스 명명 규칙

```
패턴: [리소스약어]-az01-[환경]-ab-[용도]-[시퀀스]

예시:
- Resource Group: rg-az01-prd-ab-01
- AKS: aks-az01-prd-ab-01
- PostgreSQL: pgdb-az01-prd-ab-01
- Storage: staz01prdab01
- Key Vault: kv-az01-prd-ab-01
```

---

## 7. 외부 의존성

| 의존성 | 설명 | 위험도 |
|--------|------|:------:|
| OpenAI API | LLM 에러 분석/수정 | 높음 |
| Claude API | 대체 LLM 옵션 | 중간 |
| Azure Services | 인프라 (AKS, PostgreSQL, ACR) | 높음 |
| GitHub | 소스 코드 연동, CI/CD | 중간 |
| Azure DevOps Artifacts | 사내 의존성 저장소 | 높음 |

---

## 8. 제약사항

| 제약 | 설명 |
|------|------|
| 네트워크 | 사내망 또는 VPN 필요 (Azure DevOps Artifacts 접근) |
| 빌드 도구 | Maven/Gradle만 지원 |
| Java 버전 | Java 8 → 11/17만 지원 |
| Spring 버전 | Spring Boot 2.x → 3.x만 지원 |
| 프로젝트 크기 | 최대 1GB 소스 코드 |

---

## 9. 운영 가이드

### 9.1 배포 프로세스

```
1. GitHub PR 생성
2. CI: 빌드 & 테스트
3. PR 머지 (main/dev)
4. CD: ArgoCD 자동 배포
5. Health Check 확인
```

### 9.2 장애 대응

| 상황 | 대응 |
|------|------|
| Pod CrashLoop | 로그 확인, 리소스 조정 |
| DB 연결 실패 | VNet/Subnet 확인 |
| 빌드 Job 타임아웃 | 리소스 증가, 타임아웃 조정 |
| LLM API 실패 | Fallback LLM 전환 |

### 9.3 백업 및 복구

| 항목 | 주기 | 보관 |
|------|------|------|
| PostgreSQL | 일 1회 | 7일 |
| Azure Files | 일 1회 | 7일 |
| 설정 (ConfigMap) | Git 관리 | - |

---

## 변경 이력

| 버전 | 날짜 | 작성자 | 변경 내용 |
|------|------|--------|----------|
| 1.0.0 | 2026-01-13 | - | 기존 PRD에서 분리 |
