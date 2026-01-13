# 신규 환경 배포 사전 체크리스트

> 새로운 환경(DEV/PRD)에 Auto-Builder를 배포하기 전에 완료해야 할 작업 목록

---

## 1. Azure 리소스 생성

### 1.1 Resource Group
```bash
# DEV
az group create -n rg-cloudtr-dev -l koreacentral

# PRD
az group create -n rg-cloudtr-prd -l koreacentral
```

### 1.2 AKS 클러스터
```bash
./azure/scripts/03-aks-cluster.sh dev
./azure/scripts/03-aks-cluster.sh prd
```
- [ ] AKS 클러스터 생성
- [ ] Key Vault CSI Driver 활성화
- [ ] ACR 연결 (`az aks update --attach-acr`)

### 1.3 PostgreSQL Flexible Server
```bash
./azure/scripts/04-postgresql.sh dev
./azure/scripts/04-postgresql.sh prd
```
- [ ] PostgreSQL 서버 생성
- [ ] `auto_builder` 데이터베이스 생성
- [ ] 관리자 계정 (`ab`) 생성
- [ ] AKS 서브넷에서 접근 허용 (방화벽 규칙)

### 1.4 Storage Account
```bash
./azure/scripts/05-storage.sh dev
./azure/scripts/05-storage.sh prd
```
- [ ] Storage Account 생성 (Premium_LRS, FileStorage)
- [ ] File Share 생성 (`cms-data`, 100GiB)

### 1.5 Key Vault
```bash
./azure/scripts/02a-keyvault.sh dev
./azure/scripts/02a-keyvault.sh prd
```
- [ ] Key Vault 생성
- [ ] AKS Managed Identity에 접근 권한 부여

### 1.6 Container Registry (최초 1회)
```bash
./azure/scripts/02-acr.sh
```
- [ ] ACR 생성 (공용 - PRD 리소스 그룹에)
- [ ] 각 AKS에 ACR 연결

---

## 2. Key Vault 시크릿 등록

각 환경의 Key Vault에 다음 시크릿을 등록해야 합니다.

### 필수 시크릿 목록

| 시크릿 이름 | 설명 | 예시 |
|------------|------|------|
| `DATABASE-URL` | PostgreSQL 연결 문자열 | `postgresql+asyncpg://ab:PASSWORD@psql-xxx.postgres.database.azure.com:5432/auto_builder?sslmode=require` |
| `JWT-SECRET-KEY` | JWT 서명 키 | 랜덤 문자열 (64자 이상 권장) |
| `INIT-PASSWORD` | 초기 관리자 비밀번호 | - |
| `REDIS-PASSWORD` | Redis 인증 비밀번호 | - |
| `NEXTAUTH-SECRET` | NextAuth 시크릿 | 랜덤 문자열 |
| `SERVICE-ACR-PASSWORD` | ACR 인증 비밀번호 | ACR Admin 또는 SP 비밀번호 |
| `DEVOPS-ARTIFACT-PASSWORD` | Azure Artifacts 인증 | PAT 토큰 |
| `OPENAI-KEY-1` ~ `6` | OpenAI API 키 | - |
| `CLAUDE-KEY` | Claude API 키 | - |

### 등록 명령어
```bash
# 예시
az keyvault secret set --vault-name kv-cloudtr-dev --name DATABASE-URL --value "postgresql+asyncpg://..."
az keyvault secret set --vault-name kv-cloudtr-dev --name JWT-SECRET-KEY --value "$(openssl rand -hex 32)"
az keyvault secret set --vault-name kv-cloudtr-dev --name REDIS-PASSWORD --value "your-redis-password"
# ... 나머지 시크릿들
```

---

## 3. GitHub 설정

### 3.1 Organization/Repository Secrets

| Secret 이름 | 설명 |
|------------|------|
| `AZURE_CREDENTIALS` | Azure 서비스 프린시펄 JSON |
| `CLIENT_SECRET` | Azure SP 비밀번호 |
| `BASE_ACR_PASSWORD` | Base ACR 비밀번호 |

### 3.2 Organization/Repository Variables

| Variable 이름 | 값 | 설명 |
|--------------|-----|------|
| `SERVICE_ACR_LOGIN_SERVER` | acraz01cloudtr.azurecr.io | ACR 주소 |
| `SERVICE_ACR_USERNAME` | acraz01cloudtr | ACR 사용자명 |
| `AKS_RESOURCE_GROUP_DEV` | rg-cloudtr-dev | DEV 리소스 그룹 |
| `AKS_RESOURCE_GROUP_PRD` | rg-cloudtr-prd | PRD 리소스 그룹 |
| `AKS_CLUSTER_DEV` | aks-cloudtr-dev | DEV 클러스터명 |
| `AKS_CLUSTER_PRD` | aks-cloudtr-prd | PRD 클러스터명 |
| `AZURE_KEYVAULT_DEV` | kv-cloudtr-dev | DEV Key Vault |
| `AZURE_KEYVAULT_PRD` | kv-cloudtr-prd | PRD Key Vault |
| `TENANT_ID` | (Azure AD Tenant ID) | Azure 인증 |
| `CLIENT_ID` | (Service Principal ID) | Azure 인증 |
| `SUBSCRIPTION_ID` | (Azure Subscription ID) | Azure 구독 |
| `APP_NAME` | auto-builder-cms / auto-builder-ui | 앱 이름 |
| `ARTIFACT_ORGANIZATION_NAME` | (조직명) | Azure DevOps |
| `CLOUDTR_ARTIFACT_FEED_NAME` | (피드명) | Azure Artifacts |

---

## 4. Kubernetes 사전 설정

### 4.1 Ingress Controller 설치
```bash
./azure/scripts/06-ingress-controller.sh dev
./azure/scripts/06-ingress-controller.sh prd
```
- [ ] NGINX Ingress Controller 설치
- [ ] External IP 확인

### 4.2 Namespace 생성 (CI/CD에서 자동 생성됨)
```bash
kubectl create namespace dev
kubectl create namespace prd
```

### 4.3 StorageClass 확인
```bash
kubectl get storageclass
# azurefile-csi-premium 존재 확인
```

---

## 5. ACR에 빌드 이미지 Push

K8s Job에서 사용할 기본 이미지들을 ACR에 push해야 합니다.

```bash
# ACR 로그인
az acr login --name acraz01cloudtr

# Maven 이미지
docker pull maven:3-openjdk-17
docker tag maven:3-openjdk-17 acraz01cloudtr.azurecr.io/java17-maven3.9.6:latest
docker push acraz01cloudtr.azurecr.io/java17-maven3.9.6:latest

# Gradle 이미지
docker pull gradle:8-jdk17
docker tag gradle:8-jdk17 acraz01cloudtr.azurecr.io/java17-gradle8:latest
docker push acraz01cloudtr.azurecr.io/java17-gradle8:latest

# Python 이미지
docker pull python:3.12-slim-bookworm
docker tag python:3.12-slim-bookworm acraz01cloudtr.azurecr.io/python:3.12-slim-bookworm
docker push acraz01cloudtr.azurecr.io/python:3.12-slim-bookworm
```

---

## 6. 애플리케이션 설정 확인

### 6.1 ConfigMap 값 확인

`kubernetes/base/cms/configmap.yaml` 및 `kubernetes/base/ui/configmap.yaml`에서:

- [ ] `BUILD_IMAGE_REGISTRY` → ACR 주소와 일치
- [ ] `BACKEND_CORS_ORIGINS` → 실제 도메인/IP
- [ ] `NEXTAUTH_URL` → 실제 서비스 URL
- [ ] `NEXT_PUBLIC_SERVER_URL` → 실제 API URL

### 6.2 Kustomize Overlay 확인

`kubernetes/overlays/{dev|prd}/kustomization.yaml`에서:

- [ ] `namespace` 값 확인 (dev / prd)
- [ ] `images.newName` → ACR 주소와 일치
- [ ] DB 연결 문자열 placeholder 확인

---

## 7. DNS 및 네트워크 (선택)

### 7.1 도메인 설정 (선택사항)
- [ ] DNS A 레코드 → Ingress External IP 연결
- [ ] TLS 인증서 설정 (cert-manager 또는 수동)

### 7.2 방화벽 규칙
- [ ] PostgreSQL → AKS 서브넷 허용
- [ ] Storage Account → 필요시 Private Endpoint

---

## 8. 최종 체크리스트

### Azure 리소스
- [ ] Resource Group 생성됨
- [ ] AKS 클러스터 Running 상태
- [ ] PostgreSQL 접근 가능
- [ ] Storage Account & File Share 생성됨
- [ ] Key Vault 모든 시크릿 등록됨
- [ ] ACR에 빌드 이미지 push됨

### GitHub
- [ ] 모든 Secrets 등록됨
- [ ] 모든 Variables 등록됨
- [ ] Service Principal에 Azure 권한 부여됨

### Kubernetes
- [ ] Ingress Controller 설치됨
- [ ] External IP 할당됨
- [ ] StorageClass 사용 가능

### 설정 파일
- [ ] ConfigMap 값 환경에 맞게 수정
- [ ] Kustomize overlay 확인

---

## 9. 배포 실행

모든 사전 작업 완료 후:

```bash
# 1. 클러스터 접속
az aks get-credentials -g rg-cloudtr-dev -n aks-cloudtr-dev

# 2. GitHub Actions 워크플로우 실행 (자동)
# dev 브랜치 push → DEV 배포
# main 브랜치 push → PRD 배포

# 3. 수동 배포 (필요시)
kubectl apply -k kubernetes/overlays/dev

# 4. 배포 확인
kubectl get pods -n dev
kubectl get ingress -n dev
```

---

## 10. 배포 후 검증

```bash
# 배포 상태 확인
./azure/scripts/verify-deployment.sh dev

# 또는 수동 확인
kubectl get pods -n dev
kubectl get svc -n dev
kubectl get ingress -n dev

# API Health Check
curl http://<EXTERNAL_IP>/api/health

# 로그 확인
kubectl logs -f deployment/cms-web-api -n dev
```

---

## 빠른 참조: 전체 스크립트 실행 순서

```bash
# 1. Azure 리소스 (한 번에)
./azure/scripts/setup-all.sh dev

# 또는 개별 실행
./azure/scripts/01-resource-group.sh dev
./azure/scripts/02-acr.sh                # 최초 1회
./azure/scripts/02a-keyvault.sh dev
./azure/scripts/03-aks-cluster.sh dev
./azure/scripts/04-postgresql.sh dev
./azure/scripts/05-storage.sh dev
./azure/scripts/06-ingress-controller.sh dev

# 2. Key Vault 시크릿 등록 (수동)
# ... 위의 시크릿 목록 참조

# 3. GitHub Variables 설정 (수동)
# ... 위의 변수 목록 참조

# 4. CI/CD 실행 또는 수동 배포
git push origin dev
```

---

## 예상 소요 시간

| 단계 | 소요 시간 |
|------|----------|
| Azure 리소스 생성 | 20-30분 |
| Key Vault 시크릿 등록 | 10분 |
| GitHub 설정 | 10분 |
| Ingress Controller | 5분 |
| 빌드 이미지 Push | 10분 |
| 첫 배포 & 검증 | 10-15분 |
| **총 예상** | **약 1시간~1시간 30분** |

---

*문서 작성일: 2026-01-08*
