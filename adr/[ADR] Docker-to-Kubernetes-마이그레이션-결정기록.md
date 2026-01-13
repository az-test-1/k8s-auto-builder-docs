# Architecture Decision Records (ADR)
# Docker에서 Kubernetes로의 마이그레이션

> Auto-Builder 시스템의 Docker Compose 기반 아키텍처에서 Azure Kubernetes Service(AKS)로 마이그레이션하면서 내린 주요 아키텍처 결정 사항을 기록합니다.

---

## 문서 정보

| 항목 | 내용 |
|------|------|
| 프로젝트 | Auto-Builder CMS/UI |
| 기간 | 2025.12 ~ 2026.01 |
| 상태 | 완료 (DEV/PRD 배포 완료) |
| 작성일 | 2026-01-12 |

---

## 목차

1. [ADR-001: 컨테이너 오케스트레이션 플랫폼 선택](#adr-001)
2. [ADR-002: 환경별 리소스 그룹 분리](#adr-002)
3. [ADR-003: Kubernetes 네임스페이스 전략](#adr-003)
4. [ADR-004: 데이터베이스 선택](#adr-004)
5. [ADR-005: 스토리지 솔루션](#adr-005)
6. [ADR-006: 시크릿 관리 방식](#adr-006)
7. [ADR-007: Ingress Controller 선택](#adr-007)
8. [ADR-008: 컨테이너 레지스트리 전략](#adr-008)
9. [ADR-009: CI/CD 파이프라인 및 배포 도구](#adr-009)
10. [ADR-010: 빌드 Job 실행 방식](#adr-010)
11. [ADR-011: 고가용성 전략 (HPA/PDB)](#adr-011)
12. [ADR-012: Redis 배포 방식](#adr-012)
13. [ADR-013: 로깅 및 모니터링](#adr-013)
14. [ADR-014: 네트워크 정책](#adr-014)

---

<a name="adr-001"></a>
## ADR-001: 컨테이너 오케스트레이션 플랫폼 선택

### 상태
✅ **승인됨** (2025-12)

### 컨텍스트
기존 시스템은 단일 VM에서 Docker Compose로 운영되었으나, 다음과 같은 한계에 직면:
- 단일 장애점(SPOF) 존재
- 수동 스케일링만 가능
- 무중단 배포 불가
- 리소스 활용 비효율

### 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. Docker Compose 유지** | 단순함, 학습곡선 없음 | 확장성/가용성 한계 |
| **B. Docker Swarm** | Compose와 유사, 학습 용이 | 커뮤니티 축소, 기능 제한 |
| **C. Azure Kubernetes Service (AKS)** | 관리형, Azure 통합, 생태계 | 학습곡선, 복잡성 |
| **D. Azure Container Apps** | 서버리스, 간편함 | 커스터마이징 제한 |

### 결정
**옵션 C: Azure Kubernetes Service (AKS)** 선택

### 근거
1. **관리형 서비스**: Control Plane 관리 부담 제거
2. **Azure 네이티브 통합**: Key Vault, ACR, Azure Files 등과 원활한 연동
3. **확장성**: HPA, Cluster Autoscaler 지원
4. **생태계**: Helm, Kustomize 등 풍부한 도구 지원
5. **비용 효율**: Control Plane 무료, Node만 과금
6. **향후 확장성**: 복잡한 워크로드 대응 가능

### 결과
- AKS 클러스터 2개 생성 (DEV, PRD)
- Node Pool: Standard_D2s_v3 (2 vCPU, 8GB)
- Kubernetes 버전: 1.28+

### 관련 파일
```
azure/scripts/03-aks-cluster.sh
```

---

<a name="adr-002"></a>
## ADR-002: 환경별 리소스 그룹 분리

### 상태
✅ **승인됨** (2026-01)

### 컨텍스트
초기에는 단일 리소스 그룹(`rg-cloudtr-aks`)에 DEV/PRD 클러스터를 모두 배치했으나, 관리 및 권한 분리 필요성 대두

### 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. 단일 리소스 그룹** | 관리 단순 | 권한 분리 어려움, 비용 추적 어려움 |
| **B. 환경별 리소스 그룹 분리** | 권한/비용 분리, 명확한 경계 | 관리 대상 증가 |
| **C. 서비스별 리소스 그룹** | 세밀한 제어 | 과도한 복잡성 |

### 결정
**옵션 B: 환경별 리소스 그룹 분리**

```
rg-cloudtr-dev  → DEV 환경 모든 리소스
rg-cloudtr-prd  → PRD 환경 모든 리소스
```

### 근거
1. **권한 분리**: DEV/PRD 접근 권한 별도 관리 가능
2. **비용 추적**: 환경별 비용 명확히 구분
3. **리소스 격리**: 실수로 인한 크로스 환경 영향 방지
4. **정책 적용**: 환경별 다른 Azure Policy 적용 가능

### 결과
- DEV: `rg-cloudtr-dev` (모든 DEV 리소스)
- PRD: `rg-cloudtr-prd` (모든 PRD 리소스)
- ACR만 PRD 리소스 그룹에서 공용으로 사용

### 변경된 파일
```
azure/scripts/00-env.sh
.github/workflows/ci-cd.yaml
kubernetes/overlays/*/kustomization.yaml
```

---

<a name="adr-003"></a>
## ADR-003: Kubernetes 네임스페이스 전략

### 상태
✅ **승인됨** (2026-01)

### 컨텍스트
환경(DEV/PRD)별 네임스페이스 명명 규칙 결정 필요

### 고려한 옵션

| 옵션 | 예시 | 장점 | 단점 |
|------|------|------|------|
| **A. 서비스명 포함** | `auto-builder-dev` | 명시적 | 길고 중복적 |
| **B. 환경명만** | `dev`, `prd` | 간결함 | 다른 서비스와 충돌 가능 |
| **C. 팀/프로젝트 접두사** | `cloudtr-dev` | 팀 구분 | 적당히 복잡 |

### 결정
**옵션 B: 환경명만 사용** (`dev`, `prd`)

### 근거
1. **클러스터 분리**: DEV/PRD가 별도 클러스터이므로 충돌 위험 없음
2. **간결성**: kubectl 명령어 타이핑 간소화
3. **표준화**: 많은 조직에서 사용하는 관행
4. **Kustomize 호환**: overlay 구조와 자연스럽게 매칭

### 결과
```yaml
# kubernetes/overlays/dev/kustomization.yaml
namespace: dev

# kubernetes/overlays/prd/kustomization.yaml
namespace: prd
```

### 변경된 파일
- `kubernetes/overlays/dev/kustomization.yaml`
- `kubernetes/overlays/prd/kustomization.yaml`
- 모든 문서에서 `auto-builder-dev` → `dev` 변경

---

<a name="adr-004"></a>
## ADR-004: 데이터베이스 선택

### 상태
✅ **승인됨** (2025-12)

### 컨텍스트
기존에는 Docker Compose로 PostgreSQL 컨테이너를 직접 운영. K8s 마이그레이션 시 DB 운영 방식 결정 필요

### 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. K8s 내 StatefulSet** | 완전한 제어, 비용 절감 | 운영 부담, 백업/복구 직접 구현 |
| **B. Azure Database for PostgreSQL (Single Server)** | 관리형 | 곧 지원 종료 예정 |
| **C. Azure Database for PostgreSQL (Flexible Server)** | 관리형, 최신, 유연한 설정 | 비용 |
| **D. Azure Cosmos DB (PostgreSQL)** | 글로벌 분산 | 과도한 스펙, 높은 비용 |

### 결정
**옵션 C: Azure Database for PostgreSQL Flexible Server**

### 근거
1. **관리형 서비스**: 백업, 패치, HA 자동 관리
2. **Flexible Server**: Single Server 대비 최신 기능, 더 나은 가격/성능
3. **Azure 통합**: VNet 통합, AAD 인증 지원
4. **확장성**: 필요시 스펙 변경 용이

### 결과
```
DEV: psql-cloudtr-dev.postgres.database.azure.com
PRD: psql-cloudtr-prd.postgres.database.azure.com
SKU: Burstable B1ms (1 vCore, 2GB)
Storage: 32GB
```

### 보안 설정
- AKS 서브넷에서만 접근 허용 (방화벽 규칙)
- SSL 필수 (`sslmode=require`)
- 비밀번호는 Key Vault에 저장

### 관련 파일
```
azure/scripts/04-postgresql.sh
```

---

<a name="adr-005"></a>
## ADR-005: 스토리지 솔루션

### 상태
✅ **승인됨** (2025-12)

### 컨텍스트
Auto-Builder는 프로젝트 소스 코드를 저장하고, 여러 Pod에서 동시에 접근해야 함. 적절한 K8s 스토리지 솔루션 선택 필요

### 고려한 옵션

| 옵션 | 접근 모드 | 성능 | 비용 |
|------|----------|------|------|
| **A. Azure Disk (Premium SSD)** | RWO | 높음 | 중간 |
| **B. Azure Files (Standard)** | RWX | 중간 | 낮음 |
| **C. Azure Files (Premium)** | RWX | 높음 | 높음 |
| **D. Azure NetApp Files** | RWX | 매우 높음 | 매우 높음 |
| **E. Azure Blob (NFS)** | RWX | 중간 | 낮음 |

### 결정
**옵션 C: Azure Files Premium**

### 근거
1. **RWX 지원**: 여러 Pod에서 동시 읽기/쓰기 필수 (빌드 Job + API 서버)
2. **SMB/NFS 지원**: K8s CSI 드라이버와 호환
3. **성능**: 빌드 작업 시 많은 파일 I/O 발생, Premium 성능 필요
4. **관리 용이**: Azure Portal에서 쉽게 관리

### 결과
```
DEV: stcloudtrdev (FileStorage, Premium_LRS)
PRD: stcloudtrprd (FileStorage, Premium_LRS)
File Share: cms-data (100 GiB)
```

### Kubernetes 설정
```yaml
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cms-data-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-csi-premium
  resources:
    requests:
      storage: 100Gi
```

### 관련 파일
```
azure/scripts/05-storage.sh
kubernetes/base/storage/pvc.yaml
```

---

<a name="adr-006"></a>
## ADR-006: 시크릿 관리 방식

### 상태
✅ **승인됨** (2025-12)

### 컨텍스트
DB 비밀번호, API 키 등 민감 정보를 안전하게 관리하고 Pod에 주입하는 방법 결정 필요

### 고려한 옵션

| 옵션 | 보안 | 운영 편의 | 감사 |
|------|:----:|:--------:|:----:|
| **A. K8s Secret (직접 생성)** | 낮음 | 높음 | 낮음 |
| **B. Sealed Secrets** | 중간 | 중간 | 중간 |
| **C. Azure Key Vault + CSI Driver** | 높음 | 중간 | 높음 |
| **D. HashiCorp Vault** | 높음 | 낮음 | 높음 |
| **E. External Secrets Operator** | 높음 | 높음 | 높음 |

### 결정
**옵션 C: Azure Key Vault + Secrets Store CSI Driver**
(CI/CD에서 Key Vault → K8s Secret 변환 방식 병행)

### 근거
1. **Azure 네이티브**: 추가 인프라 없이 Azure Key Vault 활용
2. **중앙 집중 관리**: 모든 시크릿을 Key Vault에서 통합 관리
3. **감사 로그**: Azure Key Vault의 접근 로그 자동 기록
4. **RBAC 통합**: Azure RBAC으로 접근 권한 관리
5. **CI/CD 연동**: GitHub Actions에서 Key Vault 참조하여 K8s Secret 생성

### 구현 방식
```
[Key Vault] → [GitHub Actions] → [K8s Secret] → [Pod]
     │              │
     │              └── az keyvault secret show
     │                  kubectl create secret
     │
     └── CSI Driver 방식 (대안)
```

### Key Vault 시크릿 목록
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

### 관련 파일
```
azure/scripts/02a-keyvault.sh
.github/workflows/ci-cd.yaml (secrets 섹션)
```

---

<a name="adr-007"></a>
## ADR-007: Ingress Controller 선택

### 상태
✅ **승인됨** (2025-12)

### 컨텍스트
외부 트래픽을 K8s 서비스로 라우팅하기 위한 Ingress Controller 선택 필요

### 고려한 옵션

| 옵션 | 기능 | 복잡도 | Azure 통합 |
|------|------|:------:|:----------:|
| **A. NGINX Ingress Controller** | 표준, 다양한 기능 | 낮음 | 중간 |
| **B. Azure Application Gateway Ingress** | WAF, Azure 네이티브 | 중간 | 높음 |
| **C. Traefik** | 자동 설정, 미들웨어 | 중간 | 낮음 |
| **D. Contour (Envoy)** | 고성능, gRPC | 높음 | 낮음 |

### 결정
**옵션 A: NGINX Ingress Controller**

### 근거
1. **업계 표준**: 가장 널리 사용되어 문서/커뮤니티 풍부
2. **단순성**: 설치/설정이 간단
3. **기능 충분**: 현재 요구사항(경로 기반 라우팅)에 충분
4. **비용**: Application Gateway 대비 저렴
5. **이식성**: 다른 클라우드로 이전 시에도 동일하게 사용 가능

### 설치 방법
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### Ingress 라우팅 규칙
```yaml
rules:
  - http:
      paths:
        - path: /api/auth
          pathType: Prefix
          backend:
            service:
              name: cms-web-ui    # NextAuth 처리
              port:
                number: 15001
        - path: /api
          pathType: Prefix
          backend:
            service:
              name: cms-web-api
              port:
                number: 18001
        - path: /mcp
          pathType: Prefix
          backend:
            service:
              name: cms-web-mcp
              port:
                number: 18002
        - path: /
          pathType: Prefix
          backend:
            service:
              name: cms-web-ui
              port:
                number: 15001
```

### 관련 파일
```
azure/scripts/06-ingress-controller.sh
kubernetes/base/ingress/ingress.yaml
```

---

<a name="adr-008"></a>
## ADR-008: 컨테이너 레지스트리 전략

### 상태
✅ **승인됨** (2025-12)

### 컨텍스트
컨테이너 이미지 저장소 선택 및 DEV/PRD 환경 공유 여부 결정 필요

### 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. Docker Hub** | 무료, 간편 | Rate Limit, 보안 우려 |
| **B. GitHub Container Registry** | GitHub 통합 | Azure 통합 약함 |
| **C. Azure Container Registry (환경별)** | 완전 격리 | 비용 증가, 이미지 중복 |
| **D. Azure Container Registry (공용)** | 비용 절감, 이미지 공유 | 권한 관리 주의 |

### 결정
**옵션 D: Azure Container Registry 공용 (단일 ACR)**

### 근거
1. **비용 효율**: ACR 하나로 DEV/PRD 모두 사용
2. **이미지 공유**: 동일 이미지를 DEV에서 테스트 후 PRD에 배포
3. **Azure 통합**: AKS와 원활한 연동 (Managed Identity)
4. **빌드 이미지 공유**: 빌드 Job용 기본 이미지 (Maven, Gradle 등) 공유

### ACR 설정
```
Name: acraz01cloudtr
SKU: Basic
Location: Korea Central
Resource Group: rg-cloudtr-prd (PRD에서 관리)
```

### AKS 연결
```bash
# 각 AKS에 ACR 연결
az aks update -n aks-cloudtr-dev -g rg-cloudtr-dev --attach-acr acraz01cloudtr
az aks update -n aks-cloudtr-prd -g rg-cloudtr-prd --attach-acr acraz01cloudtr
```

### 이미지 태그 전략
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

### 관련 파일
```
azure/scripts/02-acr.sh
```

---

<a name="adr-009"></a>
## ADR-009: CI/CD 파이프라인 및 배포 도구

### 상태
✅ **승인됨** (2025-12)

### 컨텍스트
K8s 환경에 맞는 CI/CD 파이프라인 및 매니페스트 관리 도구 선택 필요

### 고려한 옵션 (CI/CD)

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. GitHub Actions** | GitHub 통합, 무료 tier | Self-hosted runner 필요 시 비용 |
| **B. Azure DevOps Pipelines** | Azure 통합 | 별도 플랫폼 관리 |
| **C. GitLab CI** | 올인원 | GitHub에서 이전 필요 |
| **D. ArgoCD (GitOps)** | 선언적, 자동 동기화 | 추가 인프라 |

### 고려한 옵션 (매니페스트 관리)

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. 순수 YAML** | 단순 | 중복, 환경별 관리 어려움 |
| **B. Helm** | 템플릿, 패키지 관리 | 학습곡선, 복잡성 |
| **C. Kustomize** | 오버레이, kubectl 내장 | 템플릿 미지원 |
| **D. Helm + Kustomize** | 두 장점 결합 | 복잡성 증가 |

### 결정
- **CI/CD**: **GitHub Actions**
- **매니페스트 관리**: **Kustomize**

### 근거
1. **GitHub Actions**: 이미 GitHub 사용 중, 추가 도구 불필요
2. **Kustomize**:
   - kubectl 내장으로 추가 설치 불필요
   - base/overlay 구조로 환경별 설정 깔끔하게 관리
   - Helm 대비 학습곡선 낮음
   - 현재 요구사항에 충분

### 디렉토리 구조
```
kubernetes/
├── base/                 # 공통 리소스
│   ├── kustomization.yaml
│   ├── cms/
│   ├── ui/
│   ├── redis/
│   ├── storage/
│   ├── secrets/
│   └── ingress/
└── overlays/
    ├── dev/              # DEV 환경 오버라이드
    │   └── kustomization.yaml
    └── prd/              # PRD 환경 오버라이드
        └── kustomization.yaml
```

### CI/CD 흐름
```
1. Push to dev/main branch
2. GitHub Actions 트리거
3. Docker 이미지 빌드 & ACR Push
4. Key Vault에서 시크릿 조회
5. K8s Secret 생성
6. kustomize edit set image (태그 업데이트)
7. kubectl apply -k overlays/$ENV
8. kubectl rollout status (배포 확인)
```

### 관련 파일
```
.github/workflows/ci-cd.yaml
kubernetes/base/kustomization.yaml
kubernetes/overlays/*/kustomization.yaml
```

---

<a name="adr-010"></a>
## ADR-010: 빌드 Job 실행 방식

### 상태
✅ **승인됨** (2025-12)

### 컨텍스트
Auto-Builder는 사용자 프로젝트를 빌드하기 위해 동적으로 컨테이너를 생성해야 함. 기존에는 Docker-in-Docker(DinD)를 사용했으나, K8s 환경에서의 대안 필요

### 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. Docker-in-Docker (DinD)** | 기존 코드 재사용 | 보안 위험, 권한 문제 |
| **B. Docker Socket 마운트** | 단순 | 심각한 보안 위험 |
| **C. Kubernetes Jobs** | K8s 네이티브, 보안 | 코드 변경 필요 |
| **D. Kaniko** | 이미지 빌드 전용 | 범용 빌드에 부적합 |

### 결정
**옵션 C: Kubernetes Jobs**

### 근거
1. **보안**: DinD/Socket 마운트의 보안 위험 제거
2. **K8s 네이티브**: 리소스 관리, 스케줄링 활용
3. **격리**: 각 빌드가 독립된 Pod에서 실행
4. **모니터링**: K8s 표준 도구로 Job 상태 모니터링

### 구현 방식
```python
# CONTAINER_RUNTIME 환경변수로 분기
if settings.CONTAINER_RUNTIME == "kubernetes":
    manager = K8sJobManager()
else:
    manager = DockerManager()
```

### K8s Job 예시
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: tr-build-{uuid}
spec:
  activeDeadlineSeconds: 3600
  template:
    spec:
      containers:
        - name: build
          image: acraz01cloudtr.azurecr.io/java17-maven3.9.6:latest
          command: ["mvn", "compile"]
          volumeMounts:
            - name: data
              mountPath: /data/cms
          resources:
            limits:
              memory: "4Gi"
              cpu: "2"
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: cms-data-pvc
      imagePullSecrets:
        - name: acr-secret
      restartPolicy: Never
```

### 환경 변수
```
CONTAINER_RUNTIME=kubernetes
BUILD_IMAGE_REGISTRY=acraz01cloudtr.azurecr.io
K8S_NAMESPACE=dev
K8S_DATA_PVC_NAME=cms-data-pvc
K8S_DATA_MOUNT_PATH=/data/cms
K8S_JOB_MEMORY_LIMIT=4Gi
K8S_JOB_CPU_LIMIT=2
K8S_JOB_ACTIVE_DEADLINE=3600
```

### 관련 파일
```
src/app/domains/tr/tr_run/run/utils/k8s_job_manager.py
src/app/domains/tr/tr_run/run/utils/container_manager.py
kubernetes/base/cms/configmap.yaml
```

---

<a name="adr-011"></a>
## ADR-011: 고가용성 전략 (HPA/PDB)

### 상태
✅ **승인됨** (2026-01)

### 컨텍스트
프로덕션 환경에서 서비스 안정성 확보를 위한 자동 스케일링 및 중단 보호 전략 필요

### 결정
**HPA (Horizontal Pod Autoscaler) + PDB (Pod Disruption Budget) 적용**

### HPA 설정

| 대상 | Min | Max | CPU Target | 근거 |
|------|:---:|:---:|:----------:|------|
| cms-web-api | 1 | 5 | 70% | API 서버, 트래픽 변동 대응 |
| cms-celery-worker | 1 | 3 | 60% | 빌드 작업 부하에 따라 확장 |
| cms-web-ui | 1 | 3 | 70% | 프론트엔드, 보통 안정적 |

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
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### PDB 설정

| 대상 | minAvailable | 근거 |
|------|:------------:|------|
| cms-web-api | 1 | 최소 1개 Pod 항상 가동 |
| cms-celery-worker | 1 | 진행 중인 작업 보호 |
| redis | 1 | 데이터 손실 방지 |

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

### 관련 파일
```
kubernetes/base/cms/hpa.yaml
kubernetes/base/cms/pdb.yaml
kubernetes/base/redis/pdb.yaml
```

---

<a name="adr-012"></a>
## ADR-012: Redis 배포 방식

### 상태
✅ **승인됨** (2025-12)

### 컨텍스트
Celery 메시지 브로커 및 캐시로 사용되는 Redis 배포 방식 결정 필요

### 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. Azure Cache for Redis** | 관리형, HA 지원 | 비용 높음 |
| **B. K8s Deployment** | 단순 | 데이터 손실 위험 |
| **C. K8s StatefulSet** | 안정적 ID, PVC 지원 | 직접 운영 |
| **D. Redis Cluster (Helm)** | HA, 분산 | 복잡성, 과도한 스펙 |

### 결정
**옵션 C: Kubernetes StatefulSet**

### 근거
1. **비용 효율**: Azure Cache for Redis 대비 저렴
2. **현재 요구사항 충분**: 단일 인스턴스로 충분한 부하
3. **데이터 영속성**: PVC로 데이터 보존 가능
4. **단순성**: Redis Cluster까지 필요 없음

### 설정
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 1
  template:
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          args: ["--requirepass", "$(REDIS_PASSWORD)"]
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: managed-premium
        resources:
          requests:
            storage: 1Gi
```

### 향후 고려사항
- 트래픽 증가 시 Azure Cache for Redis로 마이그레이션 검토
- Redis Sentinel 또는 Cluster 구성 검토

### 관련 파일
```
kubernetes/base/redis/statefulset.yaml
kubernetes/base/redis/service.yaml
```

---

<a name="adr-013"></a>
## ADR-013: 로깅 및 모니터링

### 상태
🔄 **진행 중** (향후 개선 예정)

### 컨텍스트
K8s 환경에서의 로깅/모니터링 전략 필요

### 현재 상태 (MVP)
- **로깅**: kubectl logs로 직접 확인
- **모니터링**: kubectl get pods/hpa로 상태 확인

### 향후 계획

| 구성요소 | 계획 |
|----------|------|
| **로그 수집** | Azure Monitor Container Insights 또는 Loki |
| **메트릭** | Prometheus + Grafana |
| **알림** | Azure Monitor Alerts 또는 Alertmanager |
| **대시보드** | Grafana 대시보드 |

### 현재 구현
```bash
# 로그 확인
kubectl logs -f deployment/cms-web-api -n prd

# 상태 확인
kubectl get pods -n prd
kubectl get hpa -n prd
```

---

<a name="adr-014"></a>
## ADR-014: 네트워크 정책

### 상태
🔄 **보류** (향후 검토)

### 컨텍스트
Pod 간 네트워크 통신 제어를 위한 NetworkPolicy 적용 여부

### 현재 상태
- NetworkPolicy 미적용
- 모든 Pod 간 통신 허용

### 향후 검토 사항
```yaml
# 예시: API 서버만 DB 접근 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels:
      app: cms-web-api
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: <PostgreSQL_IP>/32
      ports:
        - protocol: TCP
          port: 5432
```

### 보류 사유
- 현재 AKS 내부 통신만 존재
- 외부 노출은 Ingress를 통해서만 가능
- 추가 복잡성 대비 보안 이점 낮음

---

## 결정 요약 매트릭스

| ADR | 결정 | 상태 | 영향도 |
|-----|------|:----:|:------:|
| ADR-001 | AKS (Azure Kubernetes Service) | ✅ | 높음 |
| ADR-002 | 환경별 리소스 그룹 분리 | ✅ | 중간 |
| ADR-003 | 단순 네임스페이스 (dev/prd) | ✅ | 낮음 |
| ADR-004 | PostgreSQL Flexible Server | ✅ | 높음 |
| ADR-005 | Azure Files Premium | ✅ | 높음 |
| ADR-006 | Key Vault + CSI Driver | ✅ | 높음 |
| ADR-007 | NGINX Ingress Controller | ✅ | 중간 |
| ADR-008 | 단일 ACR (공용) | ✅ | 중간 |
| ADR-009 | GitHub Actions + Kustomize | ✅ | 높음 |
| ADR-010 | K8s Jobs (빌드 실행) | ✅ | 높음 |
| ADR-011 | HPA + PDB | ✅ | 중간 |
| ADR-012 | Redis StatefulSet | ✅ | 중간 |
| ADR-013 | 로깅/모니터링 | 🔄 | 중간 |
| ADR-014 | NetworkPolicy | 🔄 | 낮음 |

---

## 참고 문서

- [[PRD] Auto-Builder 제품요구사항](./[PRD]%20Auto-Builder-제품요구사항.md)
- [[설계] azure-infra-resources-v2](./[설계]%20azure-infra-resources-v2.md)
- [신규환경-배포-체크리스트-v2](./신규환경-배포-체크리스트-v2.md)
- [AKS-MIGRATION-CHECKLIST](./auto-builder-cms-dev/azure/AKS-MIGRATION-CHECKLIST.md)
- [[학습] HPA](./[학습]%20HPA.md)
- [[학습] PDB](./[학습]%20PDB.md)
- [[학습] Ingress](./[학습]%20Ingress.md)

---

## 변경 이력

| 버전 | 날짜 | 변경 내용 |
|------|------|----------|
| 1.0 | 2026-01-12 | 초안 작성 - 14개 ADR 문서화 |

---

*문서 작성일: 2026-01-12*
