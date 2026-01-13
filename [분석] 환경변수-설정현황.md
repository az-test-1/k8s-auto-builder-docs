# 환경변수 설정 현황

> 프로젝트: auto-builder-cms / auto-builder-ui
> 환경: Azure Kubernetes Service (AKS)

---

## 흐름 요약

```
┌─────────────────┐     CI/CD      ┌─────────────────┐
│  Azure Key Vault │ ───────────── │   K8s Secret    │
│  (민감 정보 원본)  │              │  (런타임 주입)   │
└─────────────────┘               └─────────────────┘
                                          │
┌─────────────────┐                       ▼
│ GitHub Variables │ ───────────── ┌─────────────────┐
│  (CI/CD 설정)    │   Kustomize   │   K8s ConfigMap │
└─────────────────┘               │  (공개 설정)     │
                                  └─────────────────┘
```

---

## 1. Azure Key Vault (민감 정보)

| Key Vault Secret Name | 환경변수 | 용도 |
|----------------------|----------|------|
| `DATABASE-URL` | DATABASE_URL | PostgreSQL 연결 문자열 |
| `REDIS-PASSWORD` | REDIS_PASSWORD | Redis 비밀번호 |
| `INIT-PASSWORD` | INIT_PASSWORD | 초기 관리자 비밀번호 |
| `JWT-SECRET-KEY` | JWT_SECRET_KEY | JWT 서명 키 |
| `SERVICE-ACR-PASSWORD` | SERVICE_ACR_PASSWORD | ACR 인증 |
| `OPENAI-KEY-1` | OPENAI_KEY_1 | OpenAI API 키 |
| `OPENAI-KEY-2` | OPENAI_KEY_2 | OpenAI API 키 |
| `OPENAI-KEY-3` | OPENAI_KEY_3 | OpenAI API 키 |
| `OPENAI-KEY-4` | OPENAI_KEY_4 | OpenAI API 키 |
| `OPENAI-KEY-5` | OPENAI_KEY_5 | OpenAI API 키 |
| `OPENAI-KEY-6` | OPENAI_KEY_6 | OpenAI API 키 |
| `CLAUDE-KEY` | CLAUDE_KEY | Claude API 키 |
| `DEVOPS-ARTIFACT-PASSWORD` | DEVOPS_ARTIFACT_PASSWORD | Azure Artifacts 인증 |
| `NEXTAUTH-SECRET` | NEXTAUTH_SECRET | NextAuth 시크릿 (UI) |

---

## 2. K8s Secret (CI/CD에서 Key Vault → Secret 생성)

### db-credentials

| Key | 소스 |
|-----|------|
| DATABASE_URL | Key Vault `DATABASE-URL` |
| POSTGRES_PASSWORD | Key Vault에서 주입 |

### app-secrets

| Key | 소스 |
|-----|------|
| JWT_SECRET_KEY | Key Vault `JWT-SECRET-KEY` |
| INIT_PASSWORD | Key Vault `INIT-PASSWORD` |
| REDIS_PASSWORD | Key Vault `REDIS-PASSWORD` |
| CELERY_BROKER_URL | 조합: `redis://:${REDIS_PASSWORD}@redis-service:6379/0` |
| CELERY_RESULT_BACKEND | 조합: `redis://:${REDIS_PASSWORD}@redis-service:6379/0` |
| OPENAI_KEY_1~6 | Key Vault `OPENAI-KEY-1~6` |
| CLAUDE_KEY | Key Vault `CLAUDE-KEY` |
| AZURE_DEVOPS_PAT | Key Vault에서 주입 |

### ui-secrets

| Key | 소스 |
|-----|------|
| NEXTAUTH_SECRET | Key Vault `NEXTAUTH-SECRET` |

### acr-secret

| Key | 소스 |
|-----|------|
| .dockerconfigjson | SERVICE_ACR 인증 정보 |

---

## 3. K8s ConfigMap (공개 설정)

### cms-config

#### 기본 설정
| Key | 값 |
|-----|-----|
| PROJECT_NAME | AUTO-BUILDER-CMS |
| FASTAPI_ENV | prod |
| DEBUG | false |

#### API 설정
| Key | 값 |
|-----|-----|
| TR_API_PREFIX | /api/v1/tr |
| TR_MCP_API_URL | http://cms-web-mcp:18002/mcp |

#### 스토리지 경로
| Key | 값 |
|-----|-----|
| STORAGE_TYPE | LOCAL |
| UPLOAD_PATH | /data/cms/upload |
| TR_RUN_PATH | /data/cms/projects |
| COMMON_LIB_PATH | /data/cms/common_lib/.m2/repository |
| GRADLE_HOME_PATH | /data/cms/common_lib/.gradle |
| CUSTOM_SETTINGS_PATH | /data/cms/settings |

#### 로그 설정
| Key | 값 |
|-----|-----|
| LOG_DIR | /data/cms/logs |
| LOG_LEVEL | INFO |

#### Redis 설정
| Key | 값 |
|-----|-----|
| REDIS_HOST | redis-service |
| REDIS_PORT | 6379 |
| REDIS_DB | 0 |
| CACHE_TYPE | redis |

#### Kubernetes 설정
| Key | 값 |
|-----|-----|
| CONTAINER_RUNTIME | kubernetes |
| K8S_NAMESPACE | auto-builder |
| K8S_SERVICE_ACCOUNT | auto-builder-sa |
| K8S_JOB_MEMORY_LIMIT | 4Gi |
| K8S_JOB_MEMORY_REQUEST | 1Gi |
| K8S_JOB_CPU_LIMIT | 2 |
| K8S_JOB_CPU_REQUEST | 500m |
| K8S_JOB_ACTIVE_DEADLINE | 3600 |
| K8S_JOB_TTL_AFTER_FINISHED | 300 |
| K8S_DATA_PVC_NAME | cms-data-pvc |
| K8S_DATA_MOUNT_PATH | /data/cms |
| K8S_IMAGE_PULL_SECRETS | acr-secret |

#### Container Registry 설정 (K8s Job 빌드 이미지용)
| Key | 값 | 설명 |
|-----|-----|------|
| BUILD_IMAGE_REGISTRY | acrdemo01061855.azurecr.io | K8s Job에서 사용하는 빌드 이미지 레지스트리 |

#### 보안 설정
| Key | 값 |
|-----|-----|
| USE_CAPTCHA | false |
| CAPTCHA_LENGTH | 6 |
| CAPTCHA_EXPIRE_SECONDS | 300 |

#### JWT 설정
| Key | 값 |
|-----|-----|
| JWT_ALGORITHM | HS512 |
| ACCESS_TOKEN_EXPIRE_MINUTES | 30 |
| REFRESH_TOKEN_EXPIRE_DAYS | 7 |

#### DB 설정
| Key | 값 |
|-----|-----|
| DATABASE_SQL_SHOW | false |

#### CORS 설정
| Key | 값 |
|-----|-----|
| BACKEND_CORS_ORIGINS | http://localhost:15001,http://20.249.205.59 |

### ui-config

| Key | 값 |
|-----|-----|
| NODE_ENV | production |
| NEXTAUTH_URL | http://20.249.205.59 |
| NEXT_PUBLIC_SERVER_URL | http://20.249.205.59 |
| NEXT_PUBLIC_FILE_URL | http://20.249.205.59 |
| NEXT_SET_AUTHORIZATION | on |

---

## 4. GitHub Variables (CI/CD용)

### Organization Level Variables

| Variable | 용도 |
|----------|------|
| ARTIFACT_ORGANIZATION_NAME | Azure DevOps Organization |
| CLOUDTR_ARTIFACT_FEED_NAME | Azure Artifacts Feed |
| CLOUDTR_ARTIFACT_USER_NAME | Artifacts 사용자 |
| BASE_ACR_LOGIN_SERVER | Base ACR 주소 |
| BASE_ACR_USERNAME | Base ACR 사용자 |
| SERVICE_ACR_LOGIN_SERVER | Service ACR 주소 |
| SERVICE_ACR_USERNAME | Service ACR 사용자 |
| TENANT_ID | Azure Tenant ID |
| CLIENT_ID | Service Principal Client ID |
| SUBSCRIPTION_ID | Azure Subscription ID |
| AKS_RESOURCE_GROUP | AKS 리소스 그룹 |
| AKS_CLUSTER_PRD | PRD 클러스터 이름 |
| AKS_CLUSTER_DEV | DEV 클러스터 이름 |
| AZURE_KEYVAULT_PRD | PRD Key Vault 이름 |
| AZURE_KEYVAULT_DEV | DEV Key Vault 이름 |

### Repository Level Variables

| Variable | 용도 |
|----------|------|
| APP_NAME | 애플리케이션 이름 |

### GitHub Secrets

| Secret | 용도 |
|--------|------|
| BASE_ACR_PASSWORD | Base ACR 비밀번호 |
| CLIENT_SECRET | Service Principal 비밀 |

---

## 5. 환경별 오버라이드 (Kustomize)

### dev 환경 (`overlays/dev/kustomization.yaml`)

| 설정 | base 값 | dev 오버라이드 |
|------|---------|---------------|
| namespace | auto-builder | dev |
| FASTAPI_ENV | prod | development |
| LOG_LEVEL | INFO | DEBUG |
| DEBUG | false | true |
| K8S_NAMESPACE | auto-builder | dev |
| BUILD_IMAGE_REGISTRY | acrdemo01061855.azurecr.io | acraz01cloudtr.azurecr.io |
| PVC 크기 | 100Gi | 50Gi |
| Replica 수 | - | 각 1개 |

### prd 환경 (`overlays/prd/kustomization.yaml`)

| 설정 | base 값 | prd 오버라이드 |
|------|---------|---------------|
| namespace | auto-builder | prd |
| FASTAPI_ENV | prod | production |
| LOG_LEVEL | INFO | INFO |
| K8S_NAMESPACE | auto-builder | prd |
| BUILD_IMAGE_REGISTRY | acrdemo01061855.azurecr.io | acraz01cloudtr.azurecr.io |
| PVC 크기 | 100Gi | 100Gi |
| Replica 수 | - | 각 1개 (최소 사양) |

---

## 6. BUILD_IMAGE_REGISTRY 설명

### 용도
K8s Job에서 빌드 작업 시 사용하는 컨테이너 이미지의 레지스트리 주소

### 사용 위치

```
BUILD_IMAGE_REGISTRY = "acraz01cloudtr.azurecr.io"
    │
    ├─ config.py (Settings 클래스)
    │   └─ settings.BUILD_IMAGE_REGISTRY
    │
    ├─ init_handler.py
    │   └─ ${BUILD_IMAGE_REGISTRY}/docker.io/library/maven:3.8-openjdk-17
    │   └─ ${BUILD_IMAGE_REGISTRY}/docker.io/library/gradle:7.6-jdk17
    │
    ├─ python_migration_executor.py
    │   └─ ${BUILD_IMAGE_REGISTRY}/docker.io/library/python:3.12-slim-bookworm
    │
    └─ k8s_job_manager.py
        └─ K8s Job 생성 시 이미지 경로 prefix로 사용
```

### 이미지 경로 조합 예시

| BUILD_IMAGE_REGISTRY 설정 | 원본 이미지 | 최종 이미지 경로 |
|---------------------------|------------|-----------------|
| (빈 값) | maven:3.8-openjdk-17 | maven:3.8-openjdk-17 |
| acraz01cloudtr.azurecr.io | maven:3.8-openjdk-17 | acraz01cloudtr.azurecr.io/docker.io/library/maven:3.8-openjdk-17 |

### 왜 이름을 변경했나?

| 변경 전 | 변경 후 | 이유 |
|---------|---------|------|
| ACR_REGISTRY | BUILD_IMAGE_REGISTRY | "ACR"은 Azure 특정 용어, 실제 용도는 "빌드 이미지 레지스트리"이므로 용도 기반 명명으로 변경 |

---

## 7. 설정 관리 원칙

| 분류 | 저장소 | 예시 |
|------|--------|------|
| **유출 시 사고** | Azure Key Vault | DB 비밀번호, API 키, JWT 시크릿 |
| **유출 시 곤란** | K8s Secret | Redis 비밀번호, Celery URL |
| **유출돼도 OK** | K8s ConfigMap | 경로, 포트, 플래그 |
| **CI/CD 전용** | GitHub Variables | ACR 주소, 클러스터 이름 |

---

## 8. 관련 문서

- [ACR_REGISTRY 관리 표준](./ACR_REGISTRY-관리-표준.md)
- [CI/CD-Kustomize 역할분담](./CICD-Kustomize-역할분담.md)
- [유즈케이스 Java8to17 실행흐름](./유즈케이스-Java8to17-실행흐름.md)
