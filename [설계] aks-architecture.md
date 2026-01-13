# Auto-Builder AKS 아키텍처

## 전체 구성도

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    Azure Cloud                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         AKS Cluster (aks-cloudtr-prd)                         │  │
│  │                                                                               │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    Namespace: prd                          │  │  │
│  │  │                                                                         │  │  │
│  │  │  ┌─────────────┐   ┌─────────────┐   ┌──────────────────┐              │  │  │
│  │  │  │ cms-web-ui  │   │ cms-web-api │   │  cms-web-mcp     │              │  │  │
│  │  │  │  (Next.js)  │   │  (FastAPI)  │   │  (MCP Server)    │              │  │  │
│  │  │  │   :15001    │   │   :18001    │   │    :18002        │              │  │  │
│  │  │  └──────┬──────┘   └──────┬──────┘   └────────┬─────────┘              │  │  │
│  │  │         │                 │                   │                         │  │  │
│  │  │         │                 └─────────┬─────────┘                         │  │  │
│  │  │         │                           │                                   │  │  │
│  │  │         │                           ▼                                   │  │  │
│  │  │         │                 ┌─────────────────┐                          │  │  │
│  │  │         │                 │ cms-celery-     │                          │  │  │
│  │  │         │                 │    worker       │──────┐                   │  │  │
│  │  │         │                 │  (Task Queue)   │      │                   │  │  │
│  │  │         │                 └────────┬────────┘      │                   │  │  │
│  │  │         │                          │               │                   │  │  │
│  │  │         │                          ▼               ▼                   │  │  │
│  │  │         │                 ┌─────────────┐  ┌───────────────┐           │  │  │
│  │  │         │                 │   Redis     │  │  K8s Jobs     │           │  │  │
│  │  │         │                 │ (StatefulSet│  │ (Build/Test)  │           │  │  │
│  │  │         │                 │   :6379)    │  │               │           │  │  │
│  │  │         │                 └─────────────┘  └───────┬───────┘           │  │  │
│  │  │         │                                          │                   │  │  │
│  │  │         │                                          ▼                   │  │  │
│  │  │         │                          ┌───────────────────────────┐       │  │  │
│  │  │         │                          │     PVC: cms-data-pvc     │       │  │  │
│  │  │         │                          │    (Azure Files Share)    │       │  │  │
│  │  │         │                          │  /data/cms                │       │  │  │
│  │  │         │                          │  ├── projects/            │       │  │  │
│  │  │         │                          │  ├── upload/              │       │  │  │
│  │  │         │                          │  ├── common_lib/          │       │  │  │
│  │  │         │                          │  └── logs/                │       │  │  │
│  │  │         │                          └───────────────────────────┘       │  │  │
│  │  │         │                                                               │  │  │
│  │  └─────────┼───────────────────────────────────────────────────────────────┘  │  │
│  │            │                                                                   │  │
│  │  ┌─────────┴─────────┐                                                        │  │
│  │  │  NGINX Ingress    │                                                        │  │
│  │  │  Controller       │                                                        │  │
│  │  └─────────┬─────────┘                                                        │  │
│  │            │                                                                   │  │
│  └────────────┼───────────────────────────────────────────────────────────────────┘  │
│               │                                                                      │
│               ▼                                                                      │
│  ┌────────────────────────┐    ┌────────────────────────────────────────────────┐   │
│  │  Azure Load Balancer   │    │  Azure Database for PostgreSQL                 │   │
│  │  (Public IP)           │    │  (psql-cloudtr-prd.postgres.database.azure.com)│   │
│  │  20.249.205.59         │    │  Database: auto_builder                        │   │
│  └────────────────────────┘    └────────────────────────────────────────────────┘   │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │  Azure Container Registry (acrdemo01061855.azurecr.io)                         │ │
│  │  ├── auto-builder-cms-main:latest                                              │ │
│  │  ├── auto-builder-ui-main:latest                                               │ │
│  │  ├── java17-maven3.9.6:latest                                                  │ │
│  │  └── docker.io/library/python:3.12-slim-bookworm                               │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │  Azure Key Vault (kv-cloudtr-prd)                                              │ │
│  │  ├── database-url, jwt-secret, redis-password                                  │ │
│  │  ├── celery-broker-url, celery-result-backend                                  │ │
│  │  ├── OPENAI-KEY-1, CLAUDE-KEY                                                  │ │
│  │  └── nextauth-secret, init-password                                            │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 컴포넌트 상세

### Deployments

| 컴포넌트 | 이미지 | 포트 | 역할 |
|----------|--------|------|------|
| cms-web-ui | auto-builder-ui-main:latest | 15001 | Next.js 프론트엔드 |
| cms-web-api | auto-builder-cms-main:latest | 18001 | FastAPI 백엔드 API |
| cms-web-mcp | auto-builder-cms-main:latest | 18002 | MCP 서버 (LLM 통신) |
| cms-celery-worker | auto-builder-cms-main:latest | - | 비동기 작업 처리 |

### StatefulSet

| 컴포넌트 | 이미지 | 포트 | 역할 |
|----------|--------|------|------|
| redis | redis:7-alpine | 6379 | 캐시 & Celery Broker |

---

## 데이터 흐름

```
┌──────────┐     ┌──────────┐     ┌───────────┐     ┌─────────────┐
│  User    │────▶│  Ingress │────▶│  web-ui   │────▶│  web-api    │
│ Browser  │     │  (NGINX) │     │ (Next.js) │     │  (FastAPI)  │
└──────────┘     └──────────┘     └───────────┘     └──────┬──────┘
                                                          │
                      ┌───────────────────────────────────┘
                      │
                      ▼
              ┌───────────────┐
              │   web-mcp     │◀───────┐
              │ (MCP Server)  │        │
              └───────┬───────┘        │
                      │                │
                      ▼                │
              ┌───────────────┐        │
              │ celery-worker │────────┘
              └───────┬───────┘
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
    ┌──────────┐ ┌─────────┐ ┌──────────┐
    │  Redis   │ │ K8s Job │ │PostgreSQL│
    │ (Broker) │ │ (Build) │ │   (DB)   │
    └──────────┘ └────┬────┘ └──────────┘
                      │
                      ▼
              ┌───────────────┐
              │  Azure Files  │
              │  (PVC Mount)  │
              └───────────────┘
```

---

## RBAC 구성

```
┌─────────────────────────────────────────────────────────────┐
│                    ServiceAccount                            │
│                   auto-builder-sa                            │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    RoleBinding                               │
│               job-manager-rolebinding                        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                       Role                                   │
│                  job-manager-role                            │
├─────────────────────────────────────────────────────────────┤
│  Resources              │  Verbs                            │
│  ───────────────────────┼─────────────────────────────────  │
│  jobs, jobs/status      │  create, delete, get, list,       │
│                         │  watch, patch, update             │
│  pods                   │  get, list, watch                 │
│  pods/log               │  get, list, watch                 │
│  configmaps, secrets    │  get, list                        │
│  persistentvolumeclaims │  get, list                        │
└─────────────────────────────────────────────────────────────┘
```

---

## K8s Job 실행 흐름 (Build Step)

```
┌─────────────┐
│ celery-     │
│ worker      │
└──────┬──────┘
       │ 1. Create Job
       ▼
┌─────────────────────────────────────────────────────────────┐
│                      K8s Job                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Pod: tr-run-{run_id}-{step}                           │ │
│  │  ┌──────────────────────────────────────────────────┐  │ │
│  │  │  Container                                       │  │ │
│  │  │  Image: acrdemo01061855.azurecr.io/              │  │ │
│  │  │         java17-maven3.9.6:latest                 │  │ │
│  │  │                                                  │  │ │
│  │  │  Volume Mounts:                                  │  │ │
│  │  │  └── /data/cms (cms-data-pvc)                   │  │ │
│  │  │                                                  │  │ │
│  │  │  Working Dir: /data/cms/projects/{project_id}   │  │ │
│  │  └──────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
       │
       │ 2. Stream Logs
       ▼
┌─────────────┐
│ celery-     │
│ worker      │
└─────────────┘
       │ 3. Update Status
       ▼
┌─────────────┐
│ PostgreSQL  │
└─────────────┘
```

---

## 환경변수 흐름

```
┌────────────────────────────────────────────────────────────┐
│                     ConfigMap                               │
│                    (cms-config)                             │
├────────────────────────────────────────────────────────────┤
│  ACR_REGISTRY: "acrdemo01061855.azurecr.io"                │
│  K8S_NAMESPACE: "prd"                         │
│  K8S_SERVICE_ACCOUNT: "auto-builder-sa"                    │
│  CONTAINER_RUNTIME: "kubernetes"                           │
│  ...                                                        │
└─────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│                       Pod                                   │
│              (cms-web-api, celery-worker)                  │
├────────────────────────────────────────────────────────────┤
│  envFrom:                                                   │
│    - configMapRef: cms-config                              │
│    - secretRef: db-credentials                             │
│    - secretRef: app-secrets                                │
└─────────────────────────┬──────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│                   Python Settings                           │
│                    (config.py)                              │
├────────────────────────────────────────────────────────────┤
│  settings.ACR_REGISTRY  →  "acrdemo01061855.azurecr.io"   │
│  settings.K8S_NAMESPACE →  "prd"             │
│  ...                                                        │
└────────────────────────────────────────────────────────────┘
```

---

## Azure Key Vault 연동

### 구성 개요

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Key Vault 연동 아키텍처                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐                                                    │
│  │  Azure Key Vault│                                                    │
│  │ kv-cloudtr-prd  │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           │ RBAC: Key Vault Secrets User                                │
│           ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              CSI Secrets Store Driver                            │   │
│  │         (azurekeyvaultsecretsprovider addon)                     │   │
│  │                                                                  │   │
│  │  Identity: 0d9f965d-1523-44dc-beed-9d632b14d27b                 │   │
│  └────────┬─────────────────────────────────────────────────────────┘   │
│           │                                                             │
│           │ SecretProviderClass: azure-kv-secrets                       │
│           ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    K8s Secrets (자동 동기화)                      │   │
│  │  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐    │   │
│  │  │ db-credentials  │ │  app-secrets    │ │   ui-secrets    │    │   │
│  │  │ └─DATABASE_URL  │ │ └─JWT_SECRET_KEY│ │ └─NEXTAUTH_     │    │   │
│  │  │                 │ │ └─REDIS_PASSWORD│ │    SECRET       │    │   │
│  │  │                 │ │ └─CELERY_*      │ │                 │    │   │
│  │  │                 │ │ └─OPENAI_KEY_1  │ │                 │    │   │
│  │  │                 │ │ └─CLAUDE_KEY    │ │                 │    │   │
│  │  └─────────────────┘ └─────────────────┘ └─────────────────┘    │   │
│  └────────┬─────────────────────────────────────────────────────────┘   │
│           │                                                             │
│           │ Volume Mount: /mnt/secrets-store                            │
│           ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Pods (web-api, web-mcp, celery-worker, web-ui)                 │   │
│  │                                                                  │   │
│  │  env:                                                            │   │
│  │    DATABASE_URL: from secret db-credentials                      │   │
│  │    JWT_SECRET_KEY: from secret app-secrets                       │   │
│  │    ...                                                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Vault 시크릿 매핑

| Key Vault Secret | K8s Secret | Key | 용도 |
|------------------|------------|-----|------|
| database-url | db-credentials | DATABASE_URL | PostgreSQL 연결 |
| jwt-secret | app-secrets | JWT_SECRET_KEY | JWT 토큰 서명 |
| redis-password | app-secrets | REDIS_PASSWORD | Redis 인증 |
| init-password | app-secrets | INIT_PASSWORD | 초기 비밀번호 |
| celery-broker-url | app-secrets | CELERY_BROKER_URL | Celery Broker |
| celery-result-backend | app-secrets | CELERY_RESULT_BACKEND | Celery Backend |
| OPENAI-KEY-1 | app-secrets | OPENAI_KEY_1 | OpenAI API |
| CLAUDE-KEY | app-secrets | CLAUDE_KEY | Claude API |
| nextauth-secret | ui-secrets | NEXTAUTH_SECRET | NextAuth 서명 |

### 관련 리소스

| 리소스 | 이름 | 설명 |
|--------|------|------|
| Key Vault | kv-cloudtr-prd | 시크릿 저장소 |
| CSI Driver | azurekeyvaultsecretsprovider | AKS addon |
| SecretProviderClass | azure-kv-secrets | 동기화 설정 |
| Managed Identity | azurekeyvaultsecretsprovider-aks-cloudtr-prd | CSI 인증용 |

### 시크릿 관리 명령어

```bash
# Key Vault 시크릿 조회
az keyvault secret list --vault-name kv-cloudtr-prd --query "[].name" -o tsv

# 시크릿 값 조회
az keyvault secret show --vault-name kv-cloudtr-prd --name database-url --query "value" -o tsv

# 시크릿 추가/수정
az keyvault secret set --vault-name kv-cloudtr-prd --name my-secret --value "secret-value"

# Pod에서 마운트된 시크릿 확인
kubectl exec deployment/cms-web-api -n prd -- ls -la /mnt/secrets-store/

# K8s Secret 동기화 확인
kubectl get secrets -n prd
```

### 시크릿 분류 기준

```
┌─────────────────────────────────────────────────────────────┐
│                    시크릿 저장소 선택 기준                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  이 값이 유출되면?                                           │
│      │                                                      │
│      ├─ 회사 망함 ──────────▶ Azure Key Vault              │
│      │   (DB 접속, API 키, 인증서)                          │
│      │                                                      │
│      ├─ 좀 곤란함 ──────────▶ K8s Secret                   │
│      │   (내부 서비스 토큰)                                  │
│      │                                                      │
│      └─ 상관없음 ───────────▶ ConfigMap                    │
│          (URL, 포트, 경로)                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
