# ADR-004: 이미지 용어 정의 및 분류

## 상태
제안됨 (Proposed)

## 날짜
2026-01-12

## 컨텍스트

Auto-Builder 시스템에서 "이미지"라는 용어가 여러 맥락에서 사용되어 혼란을 야기한다:

1. Kustomize 설정의 이미지 플레이스홀더
2. 서비스 컨테이너 이미지 (ACR에 저장)
3. 빌드 도구 이미지 (maven, gradle 등)
4. 빌드 결과 이미지 (고객 애플리케이션)

각각의 역할과 용도가 다르므로 명확한 구분이 필요하다.

---

## 이미지 분류

### 1. Kustomize 이미지 플레이스홀더 (Image Placeholder)

**용도:** Kustomize의 이미지 변환기(Image Transformer)가 참조하는 이름

**위치:**
```yaml
# base/backend/deployment-web-api.yaml
spec:
  containers:
    - image: backend-image   # ← 플레이스홀더 (실제 이미지 아님)
```

**동작 원리:**
```
┌─────────────────────────────────────────────────────────────────┐
│  Base Deployment                                                │
│  image: backend-image  ←── 플레이스홀더 (참조명)                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓ kustomize build
┌─────────────────────────────────────────────────────────────────┐
│  Overlay (prd/kustomization.yaml)                               │
│  images:                                                        │
│    - name: backend-image              ←── 플레이스홀더 매칭       │
│      newName: acraz01ab01.azurecr.io/ab-backend-main            │
│      newTag: v1.2.3                                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓ 결과
┌─────────────────────────────────────────────────────────────────┐
│  최종 Deployment                                                │
│  image: acraz01ab01.azurecr.io/ab-backend-main:v1.2.3           │
└─────────────────────────────────────────────────────────────────┘
```

**현재 플레이스홀더 목록:**

| 플레이스홀더 | 용도 | 사용 Deployment |
|-------------|------|-----------------|
| `backend-image` | 백엔드 서비스 | ab-api, ab-mcp, ab-worker |
| `ui-image` | 프론트엔드 | ab-ui |

---

### 2. 서비스 컨테이너 이미지 (Service Container Image)

**용도:** K8s Pod에서 실행되는 실제 Docker 이미지

**저장소:** Azure Container Registry (ACR)

**명명 규칙:**
```
acraz01ab01.azurecr.io/ab-{서비스}-{환경}:{태그}
```

**이미지 목록:**

| 이미지명 | 설명 | 사용처 |
|---------|------|--------|
| `ab-backend-main` | 백엔드 PRD | ab-api, ab-mcp, ab-worker (prd) |
| `ab-backend-dev` | 백엔드 DEV | ab-api, ab-mcp, ab-worker (dev) |
| `ab-ui-main` | 프론트엔드 PRD | ab-ui (prd) |
| `ab-ui-dev` | 프론트엔드 DEV | ab-ui (dev) |

**특이사항:**
- 백엔드 이미지 1개가 3개 Deployment(api, mcp, worker)에서 공용 사용
- 실행 시 환경변수/command로 역할 구분

---

### 3. 빌드 도구 이미지 (Build Tool Image)

**용도:** Celery Worker가 빌드 작업 실행 시 사용하는 베이스 이미지

**관리 위치:** `base/backend/configmap-docker-images.yaml`

**이미지 목록:**

| 도구 | 베이스 이미지 | 용도 |
|------|-------------|------|
| `git` | `alpine:latest` | Git 클론, 체크아웃 |
| `maven` | `maven:3.8-openjdk-17` | Java/Maven 빌드 |
| `gradle` | `gradle:7.6-jdk17` | Java/Gradle 빌드 |
| `python` | `python:3.10-slim` | Python 빌드/실행 |
| `node` | `node:18-alpine` | Node.js 빌드 |
| `llm` | `python:3.10-slim` | LLM 관련 작업 |
| `test` | `ubuntu:22.04` | 통합 테스트 |

**동작 방식:**
```
┌─────────────────────────────────────────────────────────────────┐
│  Worker Pod                                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Celery Task: "build_maven_project"                        │  │
│  │                                                           │  │
│  │ 1. ConfigMap에서 maven 이미지 정보 읽기                     │  │
│  │ 2. K8s Job 생성 (image: maven:3.8-openjdk-17)             │  │
│  │ 3. Job에서 mvn clean package 실행                         │  │
│  │ 4. 빌드 결과물 PVC에 저장                                   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 4. 빌드 결과 이미지 (Build Output Image)

**용도:** 고객 소스코드를 빌드하여 생성된 애플리케이션 이미지

**저장소:** 고객 지정 Registry 또는 프로젝트별 ACR

**생성 과정:**
```
고객 소스코드 (Git)
       ↓
빌드 도구 이미지로 컴파일 (maven, gradle 등)
       ↓
Dockerfile로 이미지 빌드 (docker build)
       ↓
빌드 결과 이미지 (고객 Registry에 push)
```

**예시:**
```
customer-acr.azurecr.io/myapp:v1.0.0
```

---

## 이미지 흐름도

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Auto-Builder 이미지 체계                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐      │
│  │ Kustomize        │    │ Service          │    │ Build Tool       │      │
│  │ Placeholder      │───▶│ Container Image  │    │ Image            │      │
│  │                  │    │                  │    │                  │      │
│  │ backend-image    │    │ ab-backend-main  │    │ maven:3.8        │      │
│  │ ui-image         │    │ ab-ui-main       │    │ gradle:7.6       │      │
│  │                  │    │                  │    │ node:18          │      │
│  └──────────────────┘    └──────────────────┘    └────────┬─────────┘      │
│         │                        │                        │                │
│         │ (참조명)                │ (실행)                  │ (빌드 실행)    │
│         ▼                        ▼                        ▼                │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │                         Kubernetes Cluster                       │      │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐        ┌─────────┐       │      │
│  │  │ ab-api  │  │ ab-mcp  │  │ab-worker│───────▶│ K8s Job │       │      │
│  │  └─────────┘  └─────────┘  └─────────┘        └────┬────┘       │      │
│  └───────────────────────────────────────────────────│─────────────┘      │
│                                                       │                    │
│                                                       ▼                    │
│                                          ┌──────────────────┐              │
│                                          │ Build Output     │              │
│                                          │ Image            │              │
│                                          │                  │              │
│                                          │ customer-app:1.0 │              │
│                                          └──────────────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 권장 용어 사용

| 용어 | 영문 | 설명 | 예시 |
|------|------|------|------|
| 이미지 플레이스홀더 | Image Placeholder | Kustomize 참조명 | `backend-image` |
| 서비스 이미지 | Service Image | 서비스 컨테이너 이미지 | `ab-backend-main` |
| 빌드 도구 이미지 | Build Tool Image | 빌드 작업용 베이스 이미지 | `maven:3.8-openjdk-17` |
| 빌드 결과 이미지 | Build Output Image | 빌드 산출물 이미지 | `customer-app:1.0` |

---

## 파일별 이미지 관리

| 파일 | 관리 대상 |
|------|----------|
| `base/backend/deployment-*.yaml` | 이미지 플레이스홀더 |
| `overlays/{env}/backend/kustomization.yaml` | 서비스 이미지 (ACR URL) |
| `base/backend/configmap-docker-images.yaml` | 빌드 도구 이미지 |
| (런타임 생성) | 빌드 결과 이미지 |

---

## 결정

(검토 후 결정 예정)

### 검토 필요 사항

1. **용어 표준화**
   - 코드/문서에서 일관된 용어 사용 여부
   - 변수명, ConfigMap 키 등에 용어 반영

2. **ConfigMap 이름**
   - 현재: `ab-docker-images`
   - 제안: `ab-build-tool-images` (용도 명확화)

3. **문서화**
   - 개발자 온보딩 문서에 이미지 분류 설명 추가
   - 각 이미지 유형별 변경 절차 문서화

---

## 참고
- Kustomize Image Transformer: https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/images/
- ADR-003: K8s 리소스 명명 규칙 표준화
