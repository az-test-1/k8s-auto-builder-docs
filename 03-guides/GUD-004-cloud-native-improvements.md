# [학습] P0 Cloud Native 개선 작업 상세 가이드

> 작성일: 2026-01-08
> 목적: Docker → Kubernetes 마이그레이션 후 Cloud Native 원칙에 맞게 개선
> 대상: Auto-Builder CMS 프로젝트

---

## 목차

1. [작업 개요](#1-작업-개요)
2. [HPA (HorizontalPodAutoscaler) 구성](#2-hpa-horizontalpodautoscaler-구성)
3. [PodDisruptionBudget 생성](#3-poddisruptionbudget-생성)
4. [Prometheus Metrics 엔드포인트 추가](#4-prometheus-metrics-엔드포인트-추가)
5. [이미지 태그 전략 개선](#5-이미지-태그-전략-개선)
6. [전체 변경 파일 목록](#6-전체-변경-파일-목록)

---

## 1. 작업 개요

### 1.1 왜 이 작업이 필요한가?

12-Factor App과 Cloud Native 원칙 점검 결과, 다음 항목들이 **Critical (P0)** 로 분류되었습니다:

| 항목 | 문제점 | 관련 원칙 |
|------|--------|-----------|
| HPA 없음 | 트래픽 증가 시 수동 스케일링 필요 | Factor 8: Concurrency |
| PDB 없음 | 노드 유지보수 시 서비스 중단 위험 | Cloud Native: Resilience |
| Metrics 없음 | 애플리케이션 상태 모니터링 불가 | Cloud Native: Observability |
| :latest 태그 사용 | 롤백 어려움, 불변성 원칙 위반 | Factor 5: Build/Release/Run |

### 1.2 작업 범위

```
P0 (Critical) - 이번 작업 범위
├── HPA 구성 (수평 자동 스케일링)
├── PDB 생성 (Pod 가용성 보장)
├── Prometheus Metrics (관찰가능성)
└── 이미지 태그 전략 (불변 이미지)
```

---

## 2. HPA (HorizontalPodAutoscaler) 구성

### 2.1 HPA란 무엇인가?

**HorizontalPodAutoscaler (HPA)** 는 Kubernetes에서 워크로드의 리소스 사용량(CPU, Memory)을 모니터링하여 자동으로 Pod 수를 조절하는 컨트롤러입니다.

```
트래픽 증가 → CPU 사용률 상승 → HPA 감지 → Pod 추가 생성
트래픽 감소 → CPU 사용률 하락 → HPA 감지 → Pod 축소
```

### 2.2 왜 HPA가 필요한가?

**Before (문제점):**
```yaml
# 고정된 replica 수
spec:
  replicas: 1  # 트래픽이 아무리 많아도 1개
```

- 트래픽 증가 시 수동으로 `kubectl scale` 명령 실행 필요
- 야간/주말에 트래픽 급증 시 대응 불가
- 과도한 리소스 프로비저닝으로 비용 낭비

**After (개선):**
```yaml
# HPA가 자동으로 Pod 수 조절
minReplicas: 1   # 최소 1개 (비용 절약)
maxReplicas: 5   # 최대 5개 (트래픽 대응)
```

### 2.3 생성한 파일들

#### 2.3.1 API 서버 HPA

**파일:** `kubernetes/base/hpa/hpa-web-api.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cms-web-api-hpa
  labels:
    app.kubernetes.io/name: auto-builder
    app.kubernetes.io/component: backend
spec:
  # 어떤 Deployment를 스케일링할지 지정
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cms-web-api

  # 스케일링 범위
  minReplicas: 1    # 최소 Pod 수 (비용 효율)
  maxReplicas: 5    # 최대 Pod 수 (트래픽 대응)

  # 스케일링 기준 메트릭
  metrics:
    # CPU 기반 스케일링
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # CPU 70% 초과 시 스케일 아웃

    # Memory 기반 스케일링
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80  # Memory 80% 초과 시 스케일 아웃

  # 스케일링 동작 세부 설정
  behavior:
    # Scale Down 정책 (보수적)
    scaleDown:
      stabilizationWindowSeconds: 300  # 5분간 안정화 후 축소
      policies:
        - type: Percent
          value: 50              # 한 번에 최대 50%만 축소
          periodSeconds: 60
        - type: Pods
          value: 1               # 또는 1개씩만 축소
          periodSeconds: 60
      selectPolicy: Min          # 더 보수적인 정책 선택

    # Scale Up 정책 (적극적)
    scaleUp:
      stabilizationWindowSeconds: 0   # 즉시 반응
      policies:
        - type: Percent
          value: 100             # 한 번에 100%까지 증가 가능
          periodSeconds: 15
        - type: Pods
          value: 2               # 또는 2개씩 추가
          periodSeconds: 15
      selectPolicy: Max          # 더 적극적인 정책 선택
```

**핵심 설정 설명:**

| 설정 | 값 | 설명 |
|------|-----|------|
| `averageUtilization: 70` | CPU 70% | 평균 CPU가 70% 넘으면 Pod 추가 |
| `stabilizationWindowSeconds: 300` | 5분 | Scale Down 전 5분 대기 (급격한 축소 방지) |
| `stabilizationWindowSeconds: 0` | 즉시 | Scale Up은 즉시 반응 (트래픽 대응) |
| `selectPolicy: Min/Max` | 정책 선택 | Down은 보수적, Up은 적극적 |

#### 2.3.2 Celery Worker HPA

**파일:** `kubernetes/base/hpa/hpa-celery-worker.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cms-celery-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cms-celery-worker

  minReplicas: 1
  maxReplicas: 3    # Worker는 최대 3개 (장시간 태스크 특성)

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60  # Worker는 60%로 더 빠르게 반응

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600  # 10분 (장시간 태스크 고려)
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120           # 2분마다 1개씩만 축소
```

**Celery Worker 특수 설정:**
- `maxReplicas: 3` - 장시간 태스크(빌드 작업)가 많아 급격한 스케일링 불필요
- `stabilizationWindowSeconds: 600` - 10분 대기 (진행 중인 태스크 보호)
- `periodSeconds: 120` - 2분 간격으로 천천히 축소

#### 2.3.3 UI HPA

**파일:** `kubernetes/base/hpa/hpa-web-ui.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cms-web-ui-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cms-web-ui

  minReplicas: 1
  maxReplicas: 3

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**UI 특수 설정:**
- Next.js SSR은 상대적으로 가벼움
- Memory 메트릭 제외 (CPU만으로 충분)

### 2.4 Kustomization에 HPA 추가

**파일:** `kubernetes/base/hpa/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - hpa-web-api.yaml
  - hpa-celery-worker.yaml
  - hpa-web-ui.yaml
```

**파일:** `kubernetes/base/kustomization.yaml` (수정)

```yaml
resources:
  # ... 기존 리소스들 ...

  # 9. HPA (HorizontalPodAutoscaler) - 추가됨
  - hpa/
```

### 2.5 HPA 동작 확인 명령어

```bash
# HPA 상태 확인
kubectl get hpa -n auto-builder

# HPA 상세 정보
kubectl describe hpa cms-web-api-hpa -n auto-builder

# 실시간 모니터링
kubectl get hpa -n auto-builder -w

# 예상 출력:
# NAME              REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS
# cms-web-api-hpa   Deployment/cms-web-api 23%/70%   1         5         1
```

---

## 3. PodDisruptionBudget 생성

### 3.1 PDB란 무엇인가?

**PodDisruptionBudget (PDB)** 는 자발적 중단(voluntary disruption) 시 최소한의 Pod 가용성을 보장하는 정책입니다.

**자발적 중단 예시:**
- 노드 드레인 (kubectl drain)
- 클러스터 업그레이드
- Deployment 업데이트
- 노드 유지보수

### 3.2 왜 PDB가 필요한가?

**Before (문제점):**
```
노드 업그레이드 시작
  → kubectl drain node-1
  → 모든 Pod 종료됨
  → 서비스 중단!
  → 새 Pod 시작까지 대기...
```

**After (개선):**
```
노드 업그레이드 시작
  → kubectl drain node-1
  → PDB 확인: "최소 1개는 유지해야 함"
  → 다른 노드에 먼저 새 Pod 생성
  → 새 Pod Ready 확인
  → 기존 Pod 종료
  → 서비스 무중단!
```

### 3.3 생성한 파일들

#### 3.3.1 API 서버 PDB

**파일:** `kubernetes/base/pdb/pdb-web-api.yaml`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cms-web-api-pdb
  labels:
    app.kubernetes.io/name: auto-builder
    app.kubernetes.io/component: backend
spec:
  # 최소 1개 Pod는 항상 가용 상태 유지
  minAvailable: 1

  # 어떤 Pod에 적용할지 (라벨 셀렉터)
  selector:
    matchLabels:
      app: cms-web-api
```

**PDB 옵션 비교:**

| 옵션 | 설명 | 예시 |
|------|------|------|
| `minAvailable` | 최소 유지 Pod 수 | `minAvailable: 1` → 항상 1개 이상 |
| `maxUnavailable` | 최대 중단 가능 Pod 수 | `maxUnavailable: 1` → 1개만 중단 가능 |

**선택 기준:**
- `minAvailable: 1` - 소규모 서비스에 적합 (우리 케이스)
- `maxUnavailable: 25%` - 대규모 서비스에 적합

#### 3.3.2 Celery Worker PDB

**파일:** `kubernetes/base/pdb/pdb-celery-worker.yaml`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cms-celery-worker-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: cms-celery-worker
```

**Celery Worker PDB 중요성:**
- 장시간 실행 태스크 (빌드 작업 30분~1시간)
- 강제 종료 시 태스크 재시작 필요
- PDB로 최소 1개 Worker 유지 → 신규 태스크 처리 가능

#### 3.3.3 Redis PDB

**파일:** `kubernetes/base/pdb/pdb-redis.yaml`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: redis-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: redis
```

**Redis PDB 중요성:**
- Celery 브로커 역할 → Redis 중단 시 모든 태스크 처리 불가
- 세션/캐시 저장소 → Redis 중단 시 데이터 손실 가능
- 단일 인스턴스이므로 `minAvailable: 1` 필수

### 3.4 PDB 동작 확인 명령어

```bash
# PDB 상태 확인
kubectl get pdb -n auto-builder

# 예상 출력:
# NAME                    MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# cms-web-api-pdb         1               N/A               0                     1h
# cms-celery-worker-pdb   1               N/A               0                     1h
# redis-pdb               1               N/A               0                     1h

# PDB 상세 정보
kubectl describe pdb cms-web-api-pdb -n auto-builder

# 드레인 시 PDB 동작 테스트
kubectl drain node-name --ignore-daemonsets --delete-emptydir-data
# PDB가 있으면 "Cannot evict pod" 메시지 후 대기
```

---

## 4. Prometheus Metrics 엔드포인트 추가

### 4.1 Prometheus Metrics란?

**Prometheus**는 CNCF 표준 모니터링 도구로, 애플리케이션의 `/metrics` 엔드포인트에서 메트릭을 수집합니다.

```
애플리케이션 → /metrics 엔드포인트 노출
                    ↓
Prometheus Server → 주기적으로 수집 (scrape)
                    ↓
Grafana → 시각화 대시보드
```

### 4.2 왜 Metrics가 필요한가?

**Before (문제점):**
- 애플리케이션 상태 파악 불가
- "느려요" 라는 보고에 원인 분석 어려움
- 장애 발생 후에야 인지

**After (개선):**
- 실시간 요청 수, 응답 시간 모니터링
- 이상 징후 사전 감지 (알림)
- 병목 지점 식별 가능

### 4.3 수정한 파일들

#### 4.3.1 의존성 추가

**파일:** `requirements.txt`

```diff
  psycopg2-binary==2.9.10
+ prometheus-fastapi-instrumentator==7.0.0
  pydantic==2.10.4
```

**prometheus-fastapi-instrumentator 라이브러리:**
- FastAPI 전용 Prometheus 메트릭 라이브러리
- 자동으로 HTTP 요청 관련 메트릭 수집
- 최소한의 코드로 `/metrics` 엔드포인트 생성

#### 4.3.2 애플리케이션 코드 수정

**파일:** `src/app/main.py`

**변경 전:**
```python
import os
from contextlib import asynccontextmanager

import uvicorn
from fastapi import FastAPI
# ... 기존 imports ...
```

**변경 후:**
```python
import os
from contextlib import asynccontextmanager

import uvicorn
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.openapi.utils import get_openapi
from prometheus_fastapi_instrumentator import Instrumentator  # 추가

# ... 기존 imports ...
```

**Instrumentator 설정 추가:**

```python
app = create_app()

# Prometheus Metrics 설정
# /metrics 엔드포인트로 Prometheus 메트릭 노출
instrumentator = Instrumentator(
    # 상태 코드 그룹화 비활성화 (200, 201, 404 등 개별 추적)
    should_group_status_codes=False,

    # 템플릿이 없는 경로 무시 (404 등)
    should_ignore_untemplated=True,

    # 환경변수로 활성화/비활성화 가능
    should_respect_env_var=True,

    # 진행 중인 요청 수 추적
    should_instrument_requests_inprogress=True,

    # 메트릭 제외 경로 (노이즈 감소)
    excluded_handlers=["/health", "/metrics"],

    # 진행 중 요청 메트릭 이름
    inprogress_name="http_requests_inprogress",

    # 진행 중 요청에 라벨 추가
    inprogress_labels=True,
)

# FastAPI 앱에 메트릭 수집기 연결 및 /metrics 노출
instrumentator.instrument(app).expose(app, include_in_schema=True, tags=["Monitoring"])
```

**설정 옵션 상세 설명:**

| 옵션 | 값 | 설명 |
|------|-----|------|
| `should_group_status_codes` | False | 200, 201, 404 등 개별 추적 |
| `should_ignore_untemplated` | True | 정의되지 않은 경로 무시 |
| `excluded_handlers` | ["/health", "/metrics"] | 이 경로들은 메트릭에서 제외 |
| `inprogress_name` | "http_requests_inprogress" | 동시 요청 수 메트릭 이름 |

### 4.4 수집되는 메트릭 종류

```
# 요청 총 수
http_requests_total{method="GET", handler="/api/v1/users", status="200"} 1234

# 요청 처리 시간 (히스토그램)
http_request_duration_seconds_bucket{le="0.1", handler="/api/v1/users"} 1000
http_request_duration_seconds_bucket{le="0.5", handler="/api/v1/users"} 1200
http_request_duration_seconds_bucket{le="1.0", handler="/api/v1/users"} 1234

# 진행 중인 요청 수
http_requests_inprogress{method="GET", handler="/api/v1/builds"} 3

# 요청 크기 (바이트)
http_request_size_bytes_sum{handler="/api/v1/upload"} 1048576

# 응답 크기 (바이트)
http_response_size_bytes_sum{handler="/api/v1/download"} 5242880
```

### 4.5 ServiceMonitor 생성 (Prometheus Operator용)

**파일:** `kubernetes/base/monitoring/servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cms-web-api-monitor
  labels:
    app.kubernetes.io/name: auto-builder
    app.kubernetes.io/component: monitoring
    # Prometheus Operator가 이 라벨로 발견
    release: prometheus
spec:
  # 모니터링할 Service 선택
  selector:
    matchLabels:
      app: cms-web-api

  # 스크래핑 설정
  endpoints:
    - port: http           # Service의 포트 이름
      path: /metrics       # 메트릭 경로
      interval: 30s        # 30초마다 수집
      scrapeTimeout: 10s   # 타임아웃 10초

  # 네임스페이스 선택
  namespaceSelector:
    matchNames:
      - auto-builder
      - dev
      - prd
```

**ServiceMonitor 동작 원리:**
```
ServiceMonitor 생성
    ↓
Prometheus Operator가 감지
    ↓
Prometheus 설정 자동 업데이트
    ↓
해당 Service의 /metrics 스크래핑 시작
```

### 4.6 메트릭 확인 방법

```bash
# 로컬에서 /metrics 확인
curl http://localhost:18001/metrics

# Pod에서 직접 확인
kubectl exec -it deployment/cms-web-api -n auto-builder -- curl localhost:18001/metrics

# Prometheus UI에서 확인 (Prometheus가 설치된 경우)
# http://prometheus-server:9090 접속 후 쿼리:
# - http_requests_total
# - rate(http_requests_total[5m])
# - histogram_quantile(0.95, http_request_duration_seconds_bucket)
```

---

## 5. 이미지 태그 전략 개선

### 5.1 왜 :latest 태그가 문제인가?

**:latest 태그의 문제점:**

```
# 시나리오: 장애 발생 후 롤백 시도

Day 1: 배포 v1.0 (이미지: myapp:latest)
Day 2: 배포 v1.1 (이미지: myapp:latest) → 버그 발생!
Day 3: 롤백 시도...

kubectl rollout undo deployment/myapp
# 결과: 여전히 myapp:latest 사용
# 문제: latest는 항상 "최신"을 가리키므로 v1.1이 계속 배포됨!
```

**:latest 사용 시 추가 문제:**
- `imagePullPolicy: Always` 필수 → 매번 이미지 다운로드 → 느린 배포
- 어떤 버전이 실행 중인지 확인 불가
- 감사(audit) 추적 어려움

### 5.2 개선된 태그 전략

**새로운 태그 형식:**
```
{env}-{short_sha}-{timestamp}

예시:
- dev-abc1234-20260108-123456
- prod-def5678-20260108-143022
```

**장점:**
- 환경(dev/prod) 식별 가능
- Git 커밋 추적 가능 (abc1234)
- 배포 시점 확인 가능 (20260108-123456)
- 롤백 시 정확한 버전 지정 가능

### 5.3 수정한 파일들

#### 5.3.1 GitHub Actions 워크플로우

**파일:** `.github/workflows/deploy.yml`

**변경 전:**
```yaml
- name: 이미지 태그 생성
  run: |
    SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-7)
    IMAGE_TAG="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${SHORT_SHA}"

- name: Docker 빌드
  run: |
    docker build \
      -t ${{ steps.meta.outputs.image_tag }} \
      -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest \  # :latest 태그 생성
      .

- name: Docker 푸시
  run: |
    docker push ${{ steps.meta.outputs.image_tag }}
    docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest  # :latest 푸시
```

**변경 후:**
```yaml
- name: 이미지 태그 생성
  id: meta
  run: |
    SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-7)
    TIMESTAMP=$(date +%Y%m%d-%H%M%S)
    NAMESPACE=${{ steps.env.outputs.namespace }}

    # 새로운 태그 형식: {env}-{short_sha}-{timestamp}
    IMAGE_TAG="${NAMESPACE}-${SHORT_SHA}-${TIMESTAMP}"
    IMAGE_FULL="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${IMAGE_TAG}"

    echo "image_tag=$IMAGE_TAG" >> $GITHUB_OUTPUT
    echo "image_full=$IMAGE_FULL" >> $GITHUB_OUTPUT

- name: Docker 빌드
  run: |
    docker build \
      --build-arg COMMIT_SHA=${{ steps.meta.outputs.short_sha }} \
      --build-arg BUILD_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
      -t ${{ steps.meta.outputs.image_full }} \
      .
    # :latest 태그 없음!

- name: Docker 푸시
  run: |
    docker push ${{ steps.meta.outputs.image_full }}
    # :latest 푸시 없음!
```

**Kustomize를 활용한 배포:**
```yaml
- name: Kustomize 이미지 설정 및 배포
  working-directory: ./auto-builder-cms-dev/kubernetes/overlays/${{ env.NAMESPACE }}
  run: |
    # Kustomize로 이미지 태그 동적 설정
    kustomize edit set image \
      acrdemo01061855.azurecr.io/auto-builder-cms-main=${{ env.IMAGE_FULL }}

    # 배포 적용
    kustomize build . | kubectl apply -f -
```

#### 5.3.2 Deployment 파일 수정

**파일:** `kubernetes/base/cms/deployment-web-api.yaml`

**변경 전:**
```yaml
containers:
  - name: web-api
    image: acrdemo01061855.azurecr.io/auto-builder-cms-main:latest
    imagePullPolicy: Always  # 항상 이미지 다운로드
```

**변경 후:**
```yaml
containers:
  - name: web-api
    # 이미지 태그는 Kustomize overlay에서 설정
    # :latest 대신 명시적 버전 태그 사용 (CI/CD에서 자동 설정)
    image: acrdemo01061855.azurecr.io/auto-builder-cms-main:placeholder
    imagePullPolicy: IfNotPresent  # 이미 있으면 다운로드 안 함
```

**동일하게 수정된 파일들:**
- `deployment-celery-worker.yaml`
- `deployment-web-mcp.yaml`
- `ui/deployment.yaml`

#### 5.3.3 Kustomize Overlay 수정

**파일:** `kubernetes/overlays/dev/kustomization.yaml`

**변경 전:**
```yaml
images:
  - name: acrdemo01061855.azurecr.io/auto-builder-cms-main
    newName: acraz01cloudtr.azurecr.io/auto-builder-cms-dev
    newTag: latest  # :latest 사용
```

**변경 후:**
```yaml
# 이미지 설정 (CI/CD에서 동적으로 업데이트)
# newTag는 GitHub Actions에서 `kustomize edit set image`로 오버라이드
# 태그 형식: {env}-{short_sha}-{timestamp} (예: dev-abc1234-20260108-123456)
images:
  - name: acrdemo01061855.azurecr.io/auto-builder-cms-main
    newName: acraz01cloudtr.azurecr.io/auto-builder-cms-dev
    newTag: SET_BY_CI_CD  # placeholder - CI/CD가 실제 값으로 교체
```

### 5.4 Kustomize 이미지 변환 동작 원리

```
1. Base Deployment:
   image: acrdemo01061855.azurecr.io/auto-builder-cms-main:placeholder

2. Overlay kustomization.yaml:
   images:
     - name: acrdemo01061855.azurecr.io/auto-builder-cms-main
       newName: acraz01cloudtr.azurecr.io/auto-builder-cms-dev
       newTag: dev-abc1234-20260108-123456

3. Kustomize Build 결과:
   image: acraz01cloudtr.azurecr.io/auto-builder-cms-dev:dev-abc1234-20260108-123456
```

### 5.5 imagePullPolicy 설명

| 정책 | 동작 | 사용 케이스 |
|------|------|------------|
| `Always` | 항상 레지스트리에서 다운로드 | :latest 태그 사용 시 (비권장) |
| `IfNotPresent` | 로컬에 없을 때만 다운로드 | 명시적 태그 사용 시 (권장) |
| `Never` | 다운로드 안 함 | 로컬 이미지만 사용 |

**IfNotPresent의 장점:**
- 배포 속도 향상 (이미 있는 이미지 재사용)
- 네트워크 부하 감소
- 레지스트리 장애 시에도 기존 이미지로 동작

### 5.6 롤백 방법

```bash
# 이전 버전으로 롤백 (Kubernetes 기본)
kubectl rollout undo deployment/cms-web-api -n auto-builder

# 특정 버전으로 롤백
kubectl rollout undo deployment/cms-web-api --to-revision=3 -n auto-builder

# 또는 특정 이미지 태그로 직접 변경
kubectl set image deployment/cms-web-api \
  web-api=acraz01cloudtr.azurecr.io/auto-builder-cms-dev:dev-abc1234-20260108-123456 \
  -n auto-builder

# 배포 히스토리 확인
kubectl rollout history deployment/cms-web-api -n auto-builder
```

---

## 6. 전체 변경 파일 목록

### 6.1 신규 생성 파일

```
kubernetes/base/
├── hpa/
│   ├── hpa-web-api.yaml          # API 서버 HPA
│   ├── hpa-celery-worker.yaml    # Celery Worker HPA
│   ├── hpa-web-ui.yaml           # UI HPA
│   └── kustomization.yaml        # HPA Kustomization
├── pdb/
│   ├── pdb-web-api.yaml          # API 서버 PDB
│   ├── pdb-celery-worker.yaml    # Celery Worker PDB
│   ├── pdb-redis.yaml            # Redis PDB
│   └── kustomization.yaml        # PDB Kustomization
└── monitoring/
    ├── servicemonitor.yaml       # Prometheus ServiceMonitor
    └── kustomization.yaml        # Monitoring Kustomization
```

### 6.2 수정된 파일

| 파일 | 변경 내용 |
|------|-----------|
| `requirements.txt` | prometheus-fastapi-instrumentator 추가 |
| `src/app/main.py` | Prometheus Instrumentator 설정 추가 |
| `kubernetes/base/kustomization.yaml` | hpa/, pdb/ 리소스 추가 |
| `kubernetes/base/cms/deployment-web-api.yaml` | 이미지 태그 placeholder, imagePullPolicy 변경 |
| `kubernetes/base/cms/deployment-celery-worker.yaml` | 이미지 태그 placeholder, imagePullPolicy 변경 |
| `kubernetes/base/cms/deployment-web-mcp.yaml` | 이미지 태그 placeholder, imagePullPolicy 변경 |
| `kubernetes/base/ui/deployment.yaml` | 이미지 태그 placeholder, imagePullPolicy 변경 |
| `kubernetes/overlays/dev/kustomization.yaml` | newTag를 SET_BY_CI_CD로 변경 |
| `kubernetes/overlays/prd/kustomization.yaml` | newTag를 SET_BY_CI_CD로 변경 |
| `.github/workflows/deploy.yml` | 이미지 태그 전략 변경, Kustomize 배포 방식 |

### 6.3 변경 요약 다이어그램

```
┌─────────────────────────────────────────────────────────────────┐
│                        P0 개선 작업 요약                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [HPA]                    [PDB]                                 │
│  ┌─────────────┐         ┌─────────────┐                       │
│  │ web-api     │         │ web-api     │                       │
│  │ 1-5 replicas│         │ min: 1      │                       │
│  └─────────────┘         └─────────────┘                       │
│  ┌─────────────┐         ┌─────────────┐                       │
│  │ worker      │         │ worker      │                       │
│  │ 1-3 replicas│         │ min: 1      │                       │
│  └─────────────┘         └─────────────┘                       │
│  ┌─────────────┐         ┌─────────────┐                       │
│  │ ui          │         │ redis       │                       │
│  │ 1-3 replicas│         │ min: 1      │                       │
│  └─────────────┘         └─────────────┘                       │
│                                                                 │
│  [Prometheus Metrics]     [Image Tag Strategy]                  │
│  ┌─────────────────┐     ┌──────────────────────┐              │
│  │ /metrics        │     │ Before: :latest      │              │
│  │ - request count │     │ After:  dev-abc1234- │              │
│  │ - latency       │     │         20260108-... │              │
│  │ - in-progress   │     └──────────────────────┘              │
│  └─────────────────┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 부록: 유용한 명령어 모음

### HPA 관련
```bash
# HPA 목록 확인
kubectl get hpa -n auto-builder

# HPA 상세 정보
kubectl describe hpa cms-web-api-hpa -n auto-builder

# HPA 실시간 모니터링
watch kubectl get hpa -n auto-builder

# 부하 테스트로 HPA 동작 확인
kubectl run -i --tty load-generator --image=busybox -- /bin/sh
# 컨테이너 내에서:
while true; do wget -q -O- http://cms-web-api:18001/health; done
```

### PDB 관련
```bash
# PDB 목록 확인
kubectl get pdb -n auto-builder

# PDB 상세 정보
kubectl describe pdb cms-web-api-pdb -n auto-builder

# Eviction 테스트 (PDB 동작 확인)
kubectl drain <node-name> --ignore-daemonsets --dry-run=client
```

### Metrics 관련
```bash
# /metrics 엔드포인트 직접 확인
kubectl port-forward svc/cms-web-api 18001:18001 -n auto-builder
curl http://localhost:18001/metrics

# 특정 메트릭 필터링
curl http://localhost:18001/metrics | grep http_requests_total
```

### 이미지 관련
```bash
# 현재 배포된 이미지 확인
kubectl get deployment -n auto-builder -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.template.spec.containers[0].image}{"\n"}{end}'

# 배포 히스토리 확인
kubectl rollout history deployment/cms-web-api -n auto-builder

# 롤백
kubectl rollout undo deployment/cms-web-api -n auto-builder
```

---

## 참고 자료

- [Kubernetes HPA 공식 문서](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes PDB 공식 문서](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Prometheus FastAPI Instrumentator](https://github.com/trallnag/prometheus-fastapi-instrumentator)
- [12-Factor App](https://12factor.net/ko/)
- [Kustomize 이미지 변환](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/images/)
