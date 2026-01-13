# [학습] 이미지 관련 - CI/CD와 Kustomize 역할 분리

> 왜 수정했는지, 어떻게 수정했는지 자세한 설명

---

## 1. 한 줄 요약

**CI/CD는 "태그"만, Kustomize는 "이미지 이름"만 담당하도록 분리했다.**

---

## 2. 기존 문제점 (Before)

### 문제 1: CI/CD가 Git 파일을 직접 수정

```yaml
# .github/workflows/ci-cd.yaml 기존 코드
- name: Update Image Tags in Kustomization
  run: |
    cd kubernetes/overlays/${{ env.ENVIRONMENT }}

    # 문제! kustomization.yaml 파일을 직접 수정함
    yq -i '.images[0].newTag = "${{ needs.build.outputs.image_tag }}"' kustomization.yaml
```

**왜 문제인가?**

```
개발자 A: kustomization.yaml 수정 → Git push
     ↓
CI/CD: 같은 kustomization.yaml 수정 → Git 충돌!
     ↓
배포 실패 또는 수동 충돌 해결 필요
```

| 증상 | 설명 |
|------|------|
| Git 충돌 | 여러 배포가 동시에 일어나면 파일 수정이 충돌 |
| 히스토리 오염 | CI/CD 봇의 커밋이 Git 히스토리에 계속 쌓임 |
| 롤백 어려움 | 이미지 태그가 파일에 박혀있어서 롤백 시 또 파일 수정 필요 |

---

### 문제 2: 이미지 관리가 중복됨

```
┌──────────────────────────────────────────────────────────┐
│  현재 상태 (중복!)                                         │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Kustomize (kustomization.yaml)                          │
│  ┌────────────────────────────────────────┐              │
│  │ images:                                │              │
│  │   - name: old-registry/app             │              │
│  │     newName: new-registry/app          │ ← 이미지 이름 │
│  │     newTag: "20240101.1200"            │ ← 태그도 여기 │
│  └────────────────────────────────────────┘              │
│                         ↑                                │
│                         │ 수정                            │
│                         │                                │
│  CI/CD (GitHub Actions)                                  │
│  ┌────────────────────────────────────────┐              │
│  │ yq -i '.images[0].newTag = "$TAG"'     │ ← 또 태그 수정│
│  └────────────────────────────────────────┘              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**왜 문제인가?**

| 문제 | 설명 |
|------|------|
| 누가 책임자? | 태그를 Kustomize가 관리? CI/CD가 관리? 불명확 |
| 동기화 문제 | Kustomize 파일의 태그와 실제 배포된 태그가 다를 수 있음 |
| 복잡한 로직 | `if ! grep -q "images:"` 같은 조건문 필요 |

---

### 문제 3: 레지스트리 변수 중복

```
SERVICE_ACR_LOGIN_SERVER (GitHub vars)
    → CI/CD에서 이미지 빌드/푸시할 때 사용

BUILD_IMAGE_REGISTRY (ConfigMap)
    → K8s Job에서 빌드 이미지 가져올 때 사용

Kustomize images.newName
    → Deployment 이미지 교체할 때 사용

↓ 전부 같은 레지스트리를 가리키는데 3군데서 관리!
```

---

## 3. 해결 방법 (After)

### 핵심 원칙

```
┌─────────────────────────────────────────────────────────────┐
│  역할 분리                                                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Kustomize (정적, Git에 저장)                                │
│  ┌─────────────────────────────────────┐                    │
│  │ 담당:                               │                    │
│  │  - 이미지 레지스트리 이름            │                    │
│  │  - 태그는 "latest"로 고정           │                    │
│  │                                     │                    │
│  │ images:                             │                    │
│  │   - name: old-registry/app          │                    │
│  │     newName: acraz01cloudtr/app     │ ← 이름만           │
│  │     newTag: latest                  │ ← 기본값           │
│  └─────────────────────────────────────┘                    │
│                                                             │
│  CI/CD (동적, 실행 시점)                                     │
│  ┌─────────────────────────────────────┐                    │
│  │ 담당:                               │                    │
│  │  - 이미지 빌드/푸시                  │                    │
│  │  - 배포 시 태그만 변경               │                    │
│  │                                     │                    │
│  │ kubectl set image deployment/app \  │                    │
│  │   app=registry/app:20240108.1530    │ ← 태그만           │
│  └─────────────────────────────────────┘                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 수정 1: CI/CD에서 파일 수정 제거

**Before (문제)**
```yaml
# CI/CD가 파일을 직접 수정
- name: Update Image Tags
  run: |
    yq -i '.images[0].newTag = "$TAG"' kustomization.yaml
    kubectl apply -k .
```

**After (해결)**
```yaml
# CI/CD는 파일 수정 없이 kubectl로 태그만 변경
- name: Apply Kustomize
  run: kubectl apply -k kubernetes/overlays/$ENV

- name: Update Image Tags
  run: |
    kubectl set image deployment/cms-web-api \
      cms-web-api=$SERVICE_ACR/$APP_NAME:$TAG -n $NAMESPACE

    kubectl set image deployment/cms-web-mcp \
      cms-web-mcp=$SERVICE_ACR/$APP_NAME:$TAG -n $NAMESPACE

    kubectl set image deployment/cms-celery-worker \
      celery-worker=$SERVICE_ACR/$APP_NAME:$TAG -n $NAMESPACE
```

**장점**

| 항목 | 설명 |
|------|------|
| Git 충돌 없음 | 파일을 수정하지 않으니까 충돌 불가능 |
| 히스토리 깨끗 | CI/CD 봇 커밋이 없음 |
| 롤백 쉬움 | `kubectl set image`로 바로 이전 태그 지정 가능 |
| 단순함 | 조건문 로직 불필요 |

---

### 수정 2: Kustomize는 이름만 담당

**Before (문제)**
```yaml
# kustomization.yaml
images:
  - name: acrdemo01061855.azurecr.io/auto-builder-cms-main
    newName: acraz01cloudtr.azurecr.io/auto-builder-cms
    newTag: "20240107.1530"  # ← CI/CD가 계속 수정
```

**After (해결)**
```yaml
# kustomization.yaml
images:
  - name: acrdemo01061855.azurecr.io/auto-builder-cms-main
    newName: acraz01cloudtr.azurecr.io/auto-builder-cms
    newTag: latest  # ← 기본값으로 고정, CI/CD가 kubectl로 오버라이드
```

---

### 수정 3: 레지스트리 변수 용도 분리

```
┌─────────────────────────────────────────────────────────────┐
│  변수별 담당 구역 (명확하게 분리)                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SERVICE_ACR_LOGIN_SERVER (GitHub vars)                     │
│  └─ 용도: CI/CD에서 이미지 빌드/푸시                         │
│  └─ 사용: docker push $SERVICE_ACR_LOGIN_SERVER/app:tag     │
│                                                             │
│  Kustomize images.newName                                   │
│  └─ 용도: Deployment 이미지 이름 (정적)                      │
│  └─ 사용: Pod가 이 이미지를 pull                             │
│                                                             │
│  BUILD_IMAGE_REGISTRY (ConfigMap)                           │
│  └─ 용도: K8s Job에서 빌드 도구 이미지 가져올 때              │
│  └─ 사용: maven, gradle, python 이미지 경로                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**같은 레지스트리를 가리키지만 용도가 다름:**

| 변수 | 어디서 사용 | 언제 사용 | 뭘 위해 |
|------|------------|----------|--------|
| SERVICE_ACR_LOGIN_SERVER | GitHub Actions | 빌드 시 | 앱 이미지 push |
| Kustomize images | K8s Deployment | 배포 시 | 앱 이미지 pull |
| BUILD_IMAGE_REGISTRY | K8s Job (런타임) | 변환 작업 시 | 빌드 도구 이미지 pull |

---

## 4. 배포 흐름 비교

### Before (복잡)

```
1. 이미지 빌드
   └─ docker build & push

2. kustomization.yaml 수정  ← 파일 수정!
   └─ yq -i '.images[0].newTag = "..."'

3. Git commit & push (선택적)  ← Git 오염!
   └─ git add && git commit

4. kubectl apply -k
   └─ 수정된 파일로 배포
```

### After (단순)

```
1. 이미지 빌드
   └─ docker build & push

2. kubectl apply -k  ← 파일 수정 없음
   └─ 정적 Kustomize 적용

3. kubectl set image  ← 런타임에 태그만 변경
   └─ deployment 이미지 태그 업데이트

4. kubectl rollout status
   └─ 롤아웃 대기
```

---

## 5. 실제 예시

### 시나리오: 새 버전 배포

**1단계: CI/CD가 이미지 빌드**
```bash
# GitHub Actions
IMAGE_TAG="20240108.1530"
docker build -t $SERVICE_ACR/auto-builder-cms:$IMAGE_TAG .
docker push $SERVICE_ACR/auto-builder-cms:$IMAGE_TAG
```

**2단계: Kustomize 적용 (정적 매니페스트)**
```bash
kubectl apply -k kubernetes/overlays/prd
# → kustomization.yaml의 images 섹션 적용
# → newTag: latest 상태로 배포됨
```

**3단계: 이미지 태그 동적 업데이트**
```bash
kubectl set image deployment/cms-web-api \
  cms-web-api=acraz01cloudtr.azurecr.io/auto-builder-cms:20240108.1530 \
  -n prd

kubectl set image deployment/cms-web-mcp \
  cms-web-mcp=acraz01cloudtr.azurecr.io/auto-builder-cms:20240108.1530 \
  -n prd

kubectl set image deployment/cms-celery-worker \
  celery-worker=acraz01cloudtr.azurecr.io/auto-builder-cms:20240108.1530 \
  -n prd
```

**4단계: 롤아웃 확인**
```bash
kubectl rollout status deployment/cms-web-api -n prd
kubectl rollout status deployment/cms-web-mcp -n prd
kubectl rollout status deployment/cms-celery-worker -n prd
```

---

## 6. K8S_NAMESPACE 수정 이유

### 문제: base와 overlay 중복

**Before**
```yaml
# base/cms/configmap.yaml
data:
  K8S_NAMESPACE: auto-builder  # ← 기본값 정의

# overlays/dev/kustomization.yaml
patches:
  - patch: |-
      - op: replace  # ← replace인데
        path: /data/K8S_NAMESPACE
        value: dev
```

**문제점:**
- base에 값이 있으면서 overlay에서 replace
- base 값이 환경마다 다 달라서 base에 있을 이유가 없음

**After**
```yaml
# base/cms/configmap.yaml
data:
  # K8S_NAMESPACE 제거됨 (overlay에서만 정의)

# overlays/dev/kustomization.yaml
patches:
  - patch: |-
      - op: add  # ← add로 변경
        path: /data/K8S_NAMESPACE
        value: dev
```

**장점:**
- 환경별 값만 overlay에서 관리
- base는 공통 값만 보유
- 더 깔끔한 구조

---

## 7. 핵심 정리

### 역할 분담표

| 구분 | CI/CD | Kustomize |
|------|-------|-----------|
| 변경 주기 | 배포할 때마다 | 환경 설정 변경할 때만 |
| 파일 수정 | 안 함 | Git에서 관리 |
| 이미지 이름 | 참조만 | 정의 |
| 이미지 태그 | 동적 설정 | latest (기본값) |
| Secret | Key Vault → K8s | 관여 안 함 |
| ConfigMap | 관여 안 함 | 패치 |

### 기억할 것

```
1. CI/CD는 kustomization.yaml 파일을 절대 수정하지 않는다
2. 이미지 태그는 kubectl set image로 런타임에 변경한다
3. Kustomize는 정적 설정만 담당한다 (Git에 저장되는 것들)
4. 변수는 용도별로 분리한다 (빌드용 vs 배포용 vs Job용)
```

---

## 8. 관련 문서

- [CI/CD-Kustomize 역할분담](./CICD-Kustomize-역할분담.md)
- [CI/CD-Kustomize 진단결과](./CICD-Kustomize-진단결과.md)
- [환경변수 설정 현황](./환경변수-설정현황.md)
- [ACR Registry 관리 표준](./ACR_REGISTRY-관리-표준.md)
