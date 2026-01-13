# Auto-Builder 환경별 설정 현황

> 업데이트: 2026-01-08
> 상태: DEV/PRD 운영 중

---

## 1. 환경별 네임스페이스 및 리소스 그룹

| 환경 | 리소스 그룹 | AKS 클러스터 | 네임스페이스 | 브랜치 |
|:----:|:-----------:|:------------:|:------------:|:------:|
| DEV | `rg-cloudtr-dev` | aks-cloudtr-dev | `dev` | dev |
| PRD | `rg-cloudtr-prd` | aks-cloudtr-prd | `prd` | main |

---

## 2. Azure 리소스 비교

| 리소스 | DEV (rg-cloudtr-dev) | PRD (rg-cloudtr-prd) |
|--------|-----|-----|
| **AKS Cluster** | aks-cloudtr-dev | aks-cloudtr-prd |
| **PostgreSQL** | psql-cloudtr-dev | psql-cloudtr-prd |
| **Key Vault** | kv-cloudtr-dev | kv-cloudtr-prd |
| **Storage Account** | stcloudtrdev | stcloudtrprd |
| **ACR** | - | acraz01cloudtr (공용) |

---

## 3. 컨테이너 이미지

| 구분 | DEV | PRD |
|------|-----|-----|
| **CMS 이미지** | `acraz01cloudtr.azurecr.io/auto-builder-cms-dev` | `acraz01cloudtr.azurecr.io/auto-builder-cms-main` |
| **UI 이미지** | `acraz01cloudtr.azurecr.io/auto-builder-ui-dev` | `acraz01cloudtr.azurecr.io/auto-builder-ui-main` |
| **태그 형식** | dev-{sha}-{timestamp} | prod-{sha}-{timestamp} |

---

## 4. ConfigMap 환경변수 비교

### 4.1 CMS ConfigMap (`cms-config`)

| 변수명 | DEV | PRD | 비고 |
|--------|:---:|:---:|------|
| `FASTAPI_ENV` | development | production | 환경 구분 |
| `DEBUG` | true | false | 디버그 모드 |
| `LOG_LEVEL` | DEBUG | INFO | 로그 레벨 |
| `K8S_NAMESPACE` | dev | prd | K8s Job 생성 시 사용 |
| `BUILD_IMAGE_REGISTRY` | acraz01cloudtr.azurecr.io | acraz01cloudtr.azurecr.io | 동일 |

#### 공통 설정 (DEV/PRD 동일)

```yaml
# 프로젝트 설정
PROJECT_NAME: "AUTO-BUILDER-CMS"

# API 설정
TR_API_PREFIX: "/api/v1/tr"
TR_MCP_API_URL: "http://cms-web-mcp:18002/mcp"

# 스토리지 설정
STORAGE_TYPE: "AZURE_BLOB"
UPLOAD_PATH: "/data/cms/upload"
TR_RUN_PATH: "/data/cms/projects"
COMMON_LIB_PATH: "/data/cms/common_lib/.m2/repository"
GRADLE_HOME_PATH: "/data/cms/common_lib/.gradle"
CUSTOM_SETTINGS_PATH: "/data/cms/settings"

# Redis 설정
REDIS_HOST: "redis-service"
REDIS_PORT: "6379"
REDIS_DB: "0"
CACHE_TYPE: "redis"

# Kubernetes Job 설정
CONTAINER_RUNTIME: "kubernetes"
K8S_SERVICE_ACCOUNT: "auto-builder-sa"
K8S_JOB_MEMORY_LIMIT: "4Gi"
K8S_JOB_MEMORY_REQUEST: "1Gi"
K8S_JOB_CPU_LIMIT: "2"
K8S_JOB_CPU_REQUEST: "500m"
K8S_JOB_ACTIVE_DEADLINE: "3600"
K8S_JOB_TTL_AFTER_FINISHED: "300"
K8S_DATA_PVC_NAME: "cms-data-pvc"
K8S_DATA_MOUNT_PATH: "/data/cms"
K8S_IMAGE_PULL_SECRETS: "acr-secret"

# JWT 설정
JWT_ALGORITHM: "HS512"
ACCESS_TOKEN_EXPIRE_MINUTES: "30"
REFRESH_TOKEN_EXPIRE_DAYS: "7"
```

### 4.2 UI ConfigMap (`ui-config`)

| 변수명 | 값 | 비고 |
|--------|:--:|------|
| `NODE_ENV` | production | DEV/PRD 동일 |
| `NEXTAUTH_URL` | http://20.249.205.59 | External IP |
| `NEXT_PUBLIC_SERVER_URL` | http://20.249.205.59 | API 서버 URL |
| `NEXT_PUBLIC_FILE_URL` | http://20.249.205.59 | 파일 서버 URL |
| `NEXT_SET_AUTHORIZATION` | on | 인증 헤더 설정 |

---

## 5. Secret 환경변수

### 5.1 Key Vault 시크릿 (CI/CD에서 K8s Secret으로 변환)

| 시크릿 이름 | DEV Key Vault | PRD Key Vault | 용도 |
|-------------|:-------------:|:-------------:|------|
| `DATABASE-URL` | kv-cloudtr-dev | kv-cloudtr-prd | PostgreSQL 연결 |
| `JWT-SECRET-KEY` | kv-cloudtr-dev | kv-cloudtr-prd | JWT 서명 키 |
| `INIT-PASSWORD` | kv-cloudtr-dev | kv-cloudtr-prd | 초기 관리자 비밀번호 |
| `REDIS-PASSWORD` | kv-cloudtr-dev | kv-cloudtr-prd | Redis 인증 |
| `NEXTAUTH-SECRET` | kv-cloudtr-dev | kv-cloudtr-prd | NextAuth 시크릿 |
| `SERVICE-ACR-PASSWORD` | kv-cloudtr-dev | kv-cloudtr-prd | ACR 인증 |
| `OPENAI-KEY-1` ~ `6` | kv-cloudtr-dev | kv-cloudtr-prd | OpenAI API 키 |
| `CLAUDE-KEY` | kv-cloudtr-dev | kv-cloudtr-prd | Claude API 키 |

### 5.2 K8s Secret 구성

| Secret 이름 | 포함 키 | 용도 |
|-------------|---------|------|
| `db-credentials` | DATABASE_URL | DB 연결 문자열 |
| `app-secrets` | JWT_SECRET_KEY, INIT_PASSWORD, REDIS_PASSWORD, OPENAI_KEY_*, CLAUDE_KEY | 앱 시크릿 |
| `ui-secrets` | NEXTAUTH_SECRET | UI 시크릿 |
| `acr-secret` | .dockerconfigjson | ACR 이미지 Pull |

---

## 6. 리소스 사양 비교

### 6.1 Replica 수

| Deployment | DEV | PRD |
|------------|:---:|:---:|
| cms-web-api | 1 | 1 |
| cms-web-mcp | 1 | 1 |
| cms-celery-worker | 1 | 1 |
| cms-web-ui | 1 | 1 |
| redis (StatefulSet) | 1 | 1 |

### 6.2 HPA 설정

| Target | Min | Max | CPU Target |
|--------|:---:|:---:|:----------:|
| cms-web-api | 1 | 5 | 70% |
| cms-celery-worker | 1 | 3 | 60% |
| cms-web-ui | 1 | 3 | 70% |

### 6.3 PVC 용량

| PVC | DEV | PRD |
|-----|:---:|:---:|
| cms-data-pvc | 50Gi | 100Gi |

### 6.4 Redis 리소스

| 항목 | DEV | PRD |
|------|:---:|:---:|
| Memory Request | 128Mi | 256Mi |
| Memory Limit | 256Mi | 512Mi |

---

## 7. 데이터베이스 연결 정보

| 항목 | DEV | PRD |
|------|-----|-----|
| **Host** | psql-cloudtr-dev.postgres.database.azure.com | psql-cloudtr-prd.postgres.database.azure.com |
| **Port** | 5432 | 5432 |
| **Database** | auto_builder | auto_builder |
| **User** | ab | ab |
| **SSL** | require | require |

**연결 문자열 형식:**
```
postgresql+asyncpg://ab:<PASSWORD>@<HOST>:5432/auto_builder?sslmode=require
```

---

## 8. GitHub Actions 환경변수

### 8.1 Repository Variables

| 변수명 | 값 | 용도 |
|--------|:--:|------|
| `SERVICE_ACR_LOGIN_SERVER` | acraz01cloudtr.azurecr.io | ACR 주소 |
| `AKS_RESOURCE_GROUP_DEV` | rg-cloudtr-dev | DEV 리소스 그룹 |
| `AKS_RESOURCE_GROUP_PRD` | rg-cloudtr-prd | PRD 리소스 그룹 |
| `AKS_CLUSTER_PRD` | aks-cloudtr-prd | PRD 클러스터 |
| `AKS_CLUSTER_DEV` | aks-cloudtr-dev | DEV 클러스터 |
| `AZURE_KEYVAULT_PRD` | kv-cloudtr-prd | PRD Key Vault |
| `AZURE_KEYVAULT_DEV` | kv-cloudtr-dev | DEV Key Vault |

### 8.2 Repository Secrets

| 시크릿명 | 용도 |
|----------|------|
| `AZURE_CREDENTIALS` | Azure 서비스 프린시펄 |
| `SERVICE_ACR_USERNAME` | ACR 사용자명 |
| `SERVICE_ACR_PASSWORD` | ACR 비밀번호 |

---

## 9. Ingress 라우팅

| Path | Service | Port |
|------|---------|:----:|
| `/api/auth/*` | cms-web-ui | 3000 |
| `/api/*` | cms-web-api | 18001 |
| `/mcp/*` | cms-web-mcp | 18002 |
| `/files/*` | cms-web-api | 18001 |
| `/*` | cms-web-ui | 3000 |

---

## 10. 빠른 참조

### 클러스터 접속

```bash
# DEV 클러스터
az aks get-credentials -g rg-cloudtr-dev -n aks-cloudtr-dev

# PRD 클러스터
az aks get-credentials -g rg-cloudtr-prd -n aks-cloudtr-prd
```

### Pod 상태 확인

```bash
# DEV
kubectl get pods -n dev

# PRD
kubectl get pods -n prd
```

### 로그 확인

```bash
# DEV
kubectl logs -f deployment/cms-web-api -n dev

# PRD
kubectl logs -f deployment/cms-web-api -n prd
```

### Key Vault 시크릿 조회

```bash
# DEV
az keyvault secret list --vault-name kv-cloudtr-dev --query "[].name" -o tsv

# PRD
az keyvault secret list --vault-name kv-cloudtr-prd --query "[].name" -o tsv
```

---

## 관련 문서

- [DES-001-aks-architecture](../design/DES-001-aks-architecture.md)
- [ANA-006-env-variables](./ANA-006-env-variables.md)
