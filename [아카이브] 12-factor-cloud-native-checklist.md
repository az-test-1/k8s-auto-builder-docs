# 12-Factor App & Cloud Native 설계 원칙 점검 결과

> 점검일: 2026-01-08
> 대상 프로젝트: Auto-Builder CMS (Docker → Kubernetes 마이그레이션)

---

## 📋 12-Factor App 체크리스트

### Factor 1: Codebase (코드베이스)
> 버전 관리되는 하나의 코드베이스, 다수의 배포

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| Git 버전 관리 사용 | ✅ 충족 | GitHub Actions CI/CD 연동 | - |
| 단일 코드베이스 | ✅ 충족 | `auto-builder-cms-dev`, `auto-builder-ui-dev` 분리 | - |
| 환경별 배포 분리 | ✅ 충족 | `overlays/dev`, `overlays/prd` Kustomize 구조 | - |

**점수: 100%** ✅

---

### Factor 2: Dependencies (의존성)
> 명시적으로 선언하고 격리

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| 의존성 명시적 선언 | ✅ 충족 | `requirements.txt` (127개 패키지) | - |
| 버전 고정 | ⚠️ 부분충족 | 일부 패키지 버전 미고정 (`>=` 사용) | 정확한 버전 고정 필요 |
| 시스템 패키지 의존성 | ⚠️ 부분충족 | Dockerfile에서 설치하나 문서화 부족 | 시스템 의존성 문서화 |
| Private Package Index | ✅ 충족 | `pip.conf`로 Azure DevOps Artifacts 설정 | - |

**점수: 75%** ⚠️

**개선 필요사항:**
```bash
# requirements.txt에서 버전 정확히 고정
langchain>=0.3.0  →  langchain==0.3.0
```

---

### Factor 3: Config (설정)
> 환경변수에 설정 저장

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| 환경변수 기반 설정 | ✅ 충족 | Pydantic `BaseSettings` 사용 | - |
| K8s ConfigMap 사용 | ✅ 충족 | `cms-config` ConfigMap | - |
| K8s Secrets 사용 | ✅ 충족 | `db-credentials`, `app-secrets` | - |
| 하드코딩된 설정 없음 | ⚠️ 부분충족 | 일부 IP 주소 하드코딩 (`20.249.205.59`) | 외부화 필요 |
| 환경별 설정 분리 | ✅ 충족 | `.env.{local,dev,prod}` 파일 | - |
| Secret 관리 | ⚠️ 부분충족 | K8s Secrets 사용, Azure KeyVault CSI 구성됨 | KeyVault 실제 연동 필요 |

**점수: 80%** ⚠️

**개선 필요사항:**
- `BACKEND_CORS_ORIGINS`의 하드코딩된 IP 제거
- Azure KeyVault 실제 연동 활성화
- `secret-provider-class.yaml` 활용

---

### Factor 4: Backing Services (지원 서비스)
> 연결된 리소스로 취급

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| DB 외부 서비스 처리 | ✅ 충족 | Azure PostgreSQL (외부 관리형) | - |
| Redis 서비스 분리 | ✅ 충족 | Redis StatefulSet (클러스터 내부) | - |
| 연결 문자열 환경변수화 | ✅ 충족 | `DATABASE_URL`, `CELERY_BROKER_URL` | - |
| 서비스 교체 용이성 | ✅ 충족 | 환경변수 변경으로 교체 가능 | - |

**점수: 100%** ✅

---

### Factor 5: Build, Release, Run (빌드, 릴리스, 실행)
> 빌드와 실행 단계의 엄격한 분리

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| 빌드 단계 분리 | ✅ 충족 | Multi-stage Dockerfile | - |
| CI/CD 파이프라인 | ✅ 충족 | GitHub Actions `deploy.yml` | - |
| 불변 이미지 | ⚠️ 부분충족 | `:latest` 태그 사용 | 고유 태그 사용 권장 |
| 릴리스 버전 관리 | ⚠️ 부분충족 | commit SHA 태그 있으나 latest도 사용 | semantic versioning 도입 |
| 롤백 가능성 | ⚠️ 부분충족 | `imagePullPolicy: Always`로 인해 복잡 | 명시적 버전 태그 필요 |

**점수: 60%** ⚠️

**개선 필요사항:**
```yaml
# deployment에서 :latest 대신 명시적 태그 사용
image: acrdemo01061855.azurecr.io/auto-builder-cms-main:v1.2.3
# 또는
image: acrdemo01061855.azurecr.io/auto-builder-cms-main:abc1234
```

---

### Factor 6: Processes (프로세스)
> 무상태 프로세스로 실행

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| Stateless 애플리케이션 | ✅ 충족 | API, Worker 모두 Stateless | - |
| 세션 외부 저장 | ✅ 충족 | JWT 토큰 기반 인증 | - |
| 공유 파일시스템 | ⚠️ 주의필요 | PVC `ReadWriteMany` 사용 | 가능하면 Object Storage 고려 |
| 프로세스 독립성 | ✅ 충족 | 각 Pod 독립 실행 | - |

**점수: 85%** ⚠️

**개선 필요사항:**
- 파일 저장소를 Azure Blob Storage로 마이그레이션 고려
- `cms-data-pvc`를 Object Storage로 대체하면 더 Cloud Native

---

### Factor 7: Port Binding (포트 바인딩)
> 포트 바인딩을 통한 서비스 노출

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| 자체 포트 바인딩 | ✅ 충족 | FastAPI: 18001, MCP: 18002, UI: 15001 | - |
| K8s Service 사용 | ✅ 충족 | ClusterIP Services 정의됨 | - |
| Ingress 라우팅 | ✅ 충족 | NGINX Ingress Controller | - |

**점수: 100%** ✅

---

### Factor 8: Concurrency (동시성)
> 프로세스 모델을 통한 스케일 아웃

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| 수평 확장 가능 | ⚠️ 부분충족 | `replicas: 1` 고정 | HPA 설정 필요 |
| 프로세스 유형 분리 | ✅ 충족 | web-api, celery-worker, mcp 분리 | - |
| Worker 스케일링 | ⚠️ 부분충족 | Celery worker 1개 고정 | KEDA/HPA 도입 필요 |
| HPA 설정 | ❌ 미충족 | HorizontalPodAutoscaler 없음 | 필수 구현 필요 |

**점수: 50%** ❌

**개선 필요사항:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cms-web-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cms-web-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

### Factor 9: Disposability (폐기 가능성)
> 빠른 시작과 정상적인 종료

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| 빠른 시작 | ✅ 충족 | Python 앱 빠른 부팅 | - |
| Graceful Shutdown | ✅ 충족 | Celery `worker_shutdown` signal 처리 | - |
| preStop Hook | ✅ 충족 | `sleep 5` preStop hook 설정 | - |
| terminationGracePeriod | ⚠️ 부분충족 | Celery만 30초, API는 기본값 | API에도 명시 필요 |
| SIGTERM 처리 | ✅ 충족 | FastAPI lifespan, Celery signals | - |

**점수: 85%** ⚠️

**개선 필요사항:**
```yaml
# deployment-web-api.yaml에 추가
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
```

---

### Factor 10: Dev/Prod Parity (개발/프로덕션 일치)
> 개발, 스테이징, 프로덕션을 최대한 유사하게 유지

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| 동일 기술 스택 | ✅ 충족 | 모든 환경 동일 스택 | - |
| 환경별 구성 분리 | ✅ 충족 | Kustomize overlays 사용 | - |
| 동일 DB 종류 | ✅ 충족 | PostgreSQL (local/Azure) | - |
| 컨테이너 기반 개발 | ⚠️ 부분충족 | docker-compose.local.yml 있으나 복잡 | - |
| Staging 환경 | ❌ 미충족 | dev, prod만 존재 | staging 환경 추가 권장 |

**점수: 70%** ⚠️

**개선 필요사항:**
- Staging 환경 구성 추가 (`overlays/staging`)
- 프로덕션 배포 전 테스트 환경 필수화

---

### Factor 11: Logs (로그)
> 이벤트 스트림으로 취급

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| stdout 출력 | ✅ 충족 | `StreamHandler(sys.stdout)` | - |
| 구조화된 로깅 | ⚠️ 부분충족 | 텍스트 포맷, JSON 아님 | JSON 로깅 권장 |
| 중앙집중식 로깅 | ❌ 미충족 | 파일 기반 로깅 (PVC) | EFK/Loki 스택 필요 |
| 로그 집계 | ❌ 미충족 | 각 Pod 로컬 저장 | 외부 시스템 연동 필요 |
| 로그 레벨 동적 변경 | ⚠️ 부분충족 | 환경변수로 설정, 런타임 변경 불가 | - |

**점수: 40%** ❌

**개선 필요사항:**
```python
# JSON 포맷 로깅으로 변경
import json
import logging

class JSONFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "file": f"{record.filename}:{record.lineno}"
        })
```

---

### Factor 12: Admin Processes (관리 프로세스)
> 일회성 프로세스로 관리 태스크 실행

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| DB 마이그레이션 Job | ✅ 충족 | `db-init-job.yaml` 존재 | - |
| Seed Data Job | ✅ 충족 | `seed-data-job.yaml` 존재 | - |
| 동일 환경에서 실행 | ✅ 충족 | 동일 이미지, 동일 ConfigMap | - |
| 일회성 실행 보장 | ✅ 충족 | `backoffLimit: 3`, `ttlSecondsAfterFinished` | - |

**점수: 100%** ✅

---

## 📋 Cloud Native 설계 원칙 체크리스트

### 1. 컨테이너화 (Containerization)

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| Docker 컨테이너 | ✅ 충족 | 모든 서비스 컨테이너화 | - |
| Multi-stage Build | ✅ 충족 | packages → production 분리 | - |
| Non-root User | ✅ 충족 | `appuser:1001` 사용 | - |
| 최소 Base Image | ✅ 충족 | `python:3.12-slim-bookworm` | - |
| Security Context | ✅ 충족 | `allowPrivilegeEscalation: false` | - |

**점수: 100%** ✅

---

### 2. 오케스트레이션 (Orchestration)

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| Kubernetes 사용 | ✅ 충족 | AKS 배포 완료 | - |
| Deployment 사용 | ✅ 충족 | 모든 서비스 Deployment | - |
| Service Discovery | ✅ 충족 | K8s DNS 기반 서비스 발견 | - |
| RBAC 설정 | ✅ 충족 | ServiceAccount, Role, RoleBinding | - |
| Resource Limits | ✅ 충족 | requests/limits 모두 설정 | - |
| Pod Anti-Affinity | ✅ 충족 | hostname 기반 분산 | - |

**점수: 100%** ✅

---

### 3. 관찰가능성 (Observability)

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| Health Check | ✅ 충족 | `/health` 엔드포인트 | - |
| Liveness Probe | ✅ 충족 | HTTP GET 방식 | - |
| Readiness Probe | ✅ 충족 | HTTP GET 방식 | - |
| **Metrics** | ❌ 미충족 | Prometheus metrics 없음 | `/metrics` 엔드포인트 필요 |
| **Tracing** | ⚠️ 부분충족 | LangSmith만 설정 | OpenTelemetry 도입 권장 |
| **Centralized Logging** | ❌ 미충족 | 파일 기반 | EFK/Loki 스택 필요 |
| **Alerting** | ❌ 미충족 | 알림 시스템 없음 | AlertManager 연동 필요 |

**점수: 35%** ❌

**개선 필요사항:**
```python
# Prometheus metrics 추가 (prometheus-fastapi-instrumentator)
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI()
Instrumentator().instrument(app).expose(app)
```

---

### 4. 복원력 & 자가 치유 (Resilience & Self-Healing)

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| Auto-restart (K8s) | ✅ 충족 | liveness probe 실패 시 재시작 | - |
| Circuit Breaker | ❌ 미충족 | 구현 없음 | 외부 서비스 호출 시 필요 |
| Retry Logic | ⚠️ 부분충족 | Celery에만 retry 설정 | API 레벨 retry 필요 |
| Timeout 설정 | ⚠️ 부분충족 | Ingress만 설정 | 서비스간 timeout 필요 |
| PodDisruptionBudget | ❌ 미충족 | PDB 없음 | 가용성 보장 필요 |

**점수: 40%** ❌

**개선 필요사항:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cms-web-api-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: cms-web-api
```

---

### 5. 확장성 (Scalability)

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| Horizontal Scaling | ❌ 미충족 | HPA 없음 | HPA 구성 필수 |
| Vertical Scaling | ✅ 충족 | Resource limits 조정 가능 | - |
| Database Scaling | ✅ 충족 | Azure PostgreSQL 관리형 | - |
| Cache Scaling | ⚠️ 부분충족 | 단일 Redis | Redis Cluster 고려 |
| Event-Driven Scaling | ❌ 미충족 | KEDA 없음 | Celery queue 기반 스케일링 |

**점수: 40%** ❌

---

### 6. 보안 (Security)

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| Secrets Management | ⚠️ 부분충족 | K8s Secrets (base64) | Azure KeyVault 연동 필요 |
| Non-root Container | ✅ 충족 | runAsUser: 1001 | - |
| Network Policy | ❌ 미충족 | NetworkPolicy 없음 | Pod 간 통신 제한 필요 |
| Image Scanning | ❌ 미충족 | 취약점 스캔 없음 | CI/CD에 Trivy 추가 |
| TLS/HTTPS | ⚠️ 부분충족 | Ingress만 설정 | 내부 mTLS 고려 |
| RBAC 최소 권한 | ✅ 충족 | Job 관리 권한만 부여 | - |

**점수: 50%** ⚠️

**개선 필요사항:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cms-web-api-policy
spec:
  podSelector:
    matchLabels:
      app: cms-web-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: nginx-ingress
```

---

### 7. CI/CD & GitOps

| 항목 | 상태 | 현황 | 개선 필요 |
|------|------|------|-----------|
| Automated Build | ✅ 충족 | GitHub Actions | - |
| Automated Deploy | ✅ 충족 | push 기반 배포 | - |
| Environment Separation | ✅ 충족 | dev/prod 분리 | staging 추가 필요 |
| GitOps (ArgoCD/Flux) | ❌ 미충족 | kubectl 직접 사용 | ArgoCD 도입 권장 |
| Rollback Strategy | ⚠️ 부분충족 | kubectl rollout undo 가능 | 자동화 필요 |
| Canary/Blue-Green | ❌ 미충족 | 구현 없음 | 점진적 배포 전략 필요 |

**점수: 55%** ⚠️

---

## 📊 종합 점검 결과

### 12-Factor App 점수 요약

| Factor | 점수 | 상태 |
|--------|------|------|
| 1. Codebase | 100% | ✅ 충족 |
| 2. Dependencies | 75% | ⚠️ 개선필요 |
| 3. Config | 80% | ⚠️ 개선필요 |
| 4. Backing Services | 100% | ✅ 충족 |
| 5. Build/Release/Run | 60% | ⚠️ 개선필요 |
| 6. Processes | 85% | ⚠️ 개선필요 |
| 7. Port Binding | 100% | ✅ 충족 |
| 8. Concurrency | 50% | ❌ 미충족 |
| 9. Disposability | 85% | ⚠️ 개선필요 |
| 10. Dev/Prod Parity | 70% | ⚠️ 개선필요 |
| 11. Logs | 40% | ❌ 미충족 |
| 12. Admin Processes | 100% | ✅ 충족 |
| **평균** | **78.75%** | ⚠️ |

### Cloud Native 점수 요약

| 영역 | 점수 | 상태 |
|------|------|------|
| Containerization | 100% | ✅ 충족 |
| Orchestration | 100% | ✅ 충족 |
| Observability | 35% | ❌ 미충족 |
| Resilience | 40% | ❌ 미충족 |
| Scalability | 40% | ❌ 미충족 |
| Security | 50% | ⚠️ 개선필요 |
| CI/CD & GitOps | 55% | ⚠️ 개선필요 |
| **평균** | **60%** | ⚠️ |

---

## 🎯 우선순위별 개선 로드맵

### 🔴 P0 - 즉시 개선 필요 (Critical)

| # | 항목 | 관련 Factor | 작업 내용 |
|---|------|-------------|-----------|
| 1 | **HPA 구성** | Factor 8, Scalability | HorizontalPodAutoscaler 생성 |
| 2 | **Prometheus Metrics** | Observability | `/metrics` 엔드포인트 추가 |
| 3 | **이미지 태그 전략** | Factor 5 | `:latest` → semantic versioning |
| 4 | **PodDisruptionBudget** | Resilience | PDB 생성으로 가용성 보장 |

### 🟠 P1 - 단기 개선 (High Priority)

| # | 항목 | 관련 Factor | 작업 내용 |
|---|------|-------------|-----------|
| 5 | **중앙집중식 로깅** | Factor 11, Observability | JSON 로깅 + Loki/EFK 스택 |
| 6 | **Azure KeyVault 연동** | Factor 3, Security | SecretProviderClass 활성화 |
| 7 | **NetworkPolicy** | Security | Pod 간 통신 제한 |
| 8 | **Circuit Breaker** | Resilience | 외부 API 호출에 적용 |
| 9 | **Staging 환경** | Factor 10, CI/CD | `overlays/staging` 추가 -> prd, dev 두개로만갈거야 서비스가 크지않아서 |

### 🟡 P2 - 중기 개선 (Medium Priority)

| # | 항목 | 관련 Factor | 작업 내용 |
|---|------|-------------|-----------|
| 10 | **ArgoCD/Flux 도입** | CI/CD & GitOps | GitOps 기반 배포 자동화 |
| 11 | **OpenTelemetry Tracing** | Observability | 분산 트레이싱 구현 |
| 12 | **KEDA 도입** | Scalability | Celery queue 기반 스케일링 |
| 13 | **Image Vulnerability Scan** | Security | CI에 Trivy 추가 |
| 14 | **Object Storage 전환** | Factor 6 | PVC → Azure Blob Storage |

### 🟢 P3 - 장기 개선 (Nice to Have)

| # | 항목 | 관련 Factor | 작업 내용 |
|---|------|-------------|-----------|
| 15 | **Canary Deployment** | CI/CD | 점진적 배포 전략 |
| 16 | **Service Mesh (Istio)** | Security, Observability | mTLS, 고급 트래픽 관리 |
| 17 | **Redis Cluster** | Scalability | 고가용성 캐시 |
| 18 | **의존성 버전 고정** | Factor 2 | requirements.txt 정리 |

---

## 📁 생성/수정 예정 파일 목록

```
kubernetes/
├── base/
│   ├── hpa/                          # [NEW] HPA 설정
│   │   ├── hpa-web-api.yaml
│   │   ├── hpa-celery-worker.yaml
│   │   └── kustomization.yaml
│   ├── pdb/                          # [NEW] PodDisruptionBudget
│   │   ├── pdb-web-api.yaml
│   │   └── kustomization.yaml
│   ├── monitoring/                   # [NEW] 모니터링 설정
│   │   ├── servicemonitor.yaml
│   │   └── kustomization.yaml
│   ├── network-policy/               # [NEW] 네트워크 정책
│   │   ├── allow-ingress.yaml
│   │   ├── allow-redis.yaml
│   │   └── kustomization.yaml
│   └── secrets/
│       └── secret-provider-class.yaml # [MODIFY] KeyVault 연동 활성화
├── overlays/
│   └── staging/                      # [NEW] Staging 환경
│       └── kustomization.yaml
└── kustomization.yaml                # [MODIFY] 새 리소스 추가

src/app/
├── common/utils/
│   └── logger.py                     # [MODIFY] JSON 포맷 로깅
├── config/
│   └── config.py                     # [MODIFY] 하드코딩 제거
└── main.py                           # [MODIFY] Prometheus metrics 추가

.github/workflows/
└── deploy.yml                        # [MODIFY] 이미지 태그 전략, Trivy 추가
```

---

## 📌 참고 자료

- [12-Factor App](https://12factor.net/ko/)
- [CNCF Cloud Native Definition](https://github.com/cncf/toc/blob/main/DEFINITION.md)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Azure Well-Architected Framework](https://docs.microsoft.com/azure/architecture/framework/)
