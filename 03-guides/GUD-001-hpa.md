# [학습] HPA (Horizontal Pod Autoscaler)

> Pod 개수를 자동으로 늘리고 줄이는 기능

---

## 1. HPA란?

### 한 줄 정의

**CPU/메모리 사용량에 따라 Pod 개수를 자동으로 조절하는 Kubernetes 기능**

### 시각적 이해

```
트래픽 적음                         트래픽 많음
     │                                  │
     ▼                                  ▼
 ┌───────┐                    ┌───────┐ ┌───────┐ ┌───────┐
 │ Pod 1 │       자동 확장     │ Pod 1 │ │ Pod 2 │ │ Pod 3 │
 └───────┘      ──────────►   └───────┘ └───────┘ └───────┘
     │                                  │
     │                                  │
 CPU: 20%                           CPU: 70%

        ┌─────────────────────────────────────┐
        │  HPA Controller                     │
        │  "CPU 70% 넘었네? Pod 추가해야지"    │
        └─────────────────────────────────────┘
```

---

## 2. 핵심 개념

### Horizontal vs Vertical

| 구분 | Horizontal (HPA) | Vertical (VPA) |
|------|-----------------|----------------|
| 방식 | Pod **개수** 늘림 | Pod **스펙** 늘림 |
| 예시 | 1개 → 3개 | CPU 100m → 500m |
| 장점 | 빠른 확장, 무중단 | 단순 |
| 단점 | Stateless 앱에 적합 | Pod 재시작 필요 |

**우리는 HPA 사용** → Pod 개수로 확장

### HPA 동작 원리

```
┌─────────────────────────────────────────────────────────────┐
│                      HPA Controller                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Metrics Server에서 현재 CPU/메모리 사용량 수집           │
│         ↓                                                   │
│  2. 목표값(예: CPU 70%)과 현재값 비교                        │
│         ↓                                                   │
│  3. 필요 Pod 수 계산                                        │
│         ↓                                                   │
│  4. Deployment의 replicas 자동 조정                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Pod 수 계산 공식

```
필요 Pod 수 = ceil(현재 Pod 수 × (현재 사용량 / 목표 사용량))

예시:
- 현재: Pod 2개, CPU 평균 80%
- 목표: CPU 50%
- 계산: 2 × (80/50) = 3.2 → 올림 → 4개
```

---

## 3. HPA 없을 때 단점

### 현재 상태 (고정 replicas)

```yaml
# 현재 Deployment
spec:
  replicas: 1  # ← 항상 1개 고정
```

### 문제 시나리오

```
시간대별 트래픽 변화:

09:00 출근   ████████████████████ 100명 → Pod 1개 → 느려짐 ❌
12:00 점심   ████████             40명  → Pod 1개 → 여유
14:00 업무   ██████████████████   90명  → Pod 1개 → 느려짐 ❌
18:00 퇴근   ██████████████       70명  → Pod 1개 → 버벅 ⚠️
22:00 야간   ███                  15명  → Pod 1개 → 과잉 💰
```

### 구체적인 단점

| 단점 | 설명 | 영향 |
|------|------|------|
| **트래픽 급증 대응 불가** | 사용자 몰리면 Pod 1개로 감당 불가 | 응답 지연, 504 에러 |
| **수동 대응 필요** | 개발자가 직접 replicas 수정 필요 | 대응 시간 5~10분 |
| **비효율적 비용** | 야간에도 동일 리소스 유지 | 불필요한 비용 발생 |
| **장애 예방 불가** | CPU 100% 되어야 인지 | 이미 장애 발생 후 |

### 실제 장애 시나리오

```
1. 오전 9시: 동시 접속 급증
   └─ Pod 1개 CPU: 30% → 60% → 90% → 100%

2. CPU 100% 도달
   └─ 응답 시간: 200ms → 2초 → 10초 → Timeout

3. 알람 발생 → 개발자 인지
   └─ 시간 소요: 5분

4. 수동으로 replicas: 3 변경 → 배포
   └─ 시간 소요: 5분

5. 총 장애 시간: 10분+
   └─ 사용자 불만, 업무 지연
```

---

## 4. HPA 도입 시 장점

### HPA 적용 후

```
시간대별 자동 대응:

09:00 출근   ████████████████████ 100명 → Pod 3개 → 정상 ✅
12:00 점심   ████████             40명  → Pod 2개 → 자동 축소
14:00 업무   ██████████████████   90명  → Pod 3개 → 자동 확장
18:00 퇴근   ██████████████       70명  → Pod 2개 → 자동 축소
22:00 야간   ███                  15명  → Pod 1개 → 비용 절감 💰
```

### 구체적인 장점

| 장점 | 설명 | 효과 |
|------|------|------|
| **자동 확장** | 트래픽 증가 감지 → 자동 Pod 추가 | 무중단 서비스 |
| **자동 축소** | 트래픽 감소 감지 → 자동 Pod 제거 | 비용 30~50% 절감 |
| **선제 대응** | CPU 70%에서 미리 확장 | 장애 예방 |
| **운영 부담 감소** | 수동 개입 불필요 | 개발자 시간 절약 |

### HPA 동작 시나리오

```
1. 오전 9시: 동시 접속 급증
   └─ Pod 1개 CPU: 30% → 60% → 70%

2. HPA 감지: CPU 70% 초과
   └─ 자동으로 Pod 2개로 확장 (약 15초)

3. CPU 분산: 각 Pod 35%
   └─ 정상 응답 유지

4. 트래픽 계속 증가
   └─ 필요시 Pod 3개 → 4개 → 5개(max)

5. 장애 시간: 0분
   └─ 사용자 영향 없음
```

---

## 5. 현재 프로젝트 HPA 설정

### 파일 구조

```
kubernetes/
├── base/
│   ├── hpa/                          # ← HPA 설정 폴더
│   │   ├── kustomization.yaml
│   │   ├── hpa-web-api.yaml          # API 서버 HPA
│   │   ├── hpa-celery-worker.yaml    # Worker HPA
│   │   └── hpa-web-ui.yaml           # UI HPA
│   └── kustomization.yaml            # ← hpa/ 포함됨
└── overlays/
    ├── dev/kustomization.yaml
    └── prd/kustomization.yaml
```

### HPA 설정 상세

#### cms-web-api (API 서버)

```yaml
# kubernetes/base/hpa/hpa-web-api.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cms-web-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cms-web-api           # 대상 Deployment

  minReplicas: 1                # 최소 Pod 수
  maxReplicas: 5                # 최대 Pod 수

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # CPU 70% 기준
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80   # Memory 80% 기준

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # 축소 전 5분 대기
      policies:
        - type: Percent
          value: 50                      # 최대 50%씩 축소
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0     # 즉시 확장
      policies:
        - type: Pods
          value: 2                       # 최대 2개씩 확장
          periodSeconds: 15
```

**설정 의미:**

| 설정 | 값 | 의미 |
|------|-----|------|
| minReplicas | 1 | 트래픽 없어도 최소 1개 유지 |
| maxReplicas | 5 | 아무리 바빠도 5개까지만 |
| CPU 70% | 확장 트리거 | CPU 70% 넘으면 Pod 추가 |
| Memory 80% | 확장 트리거 | 메모리 80% 넘으면 Pod 추가 |
| scaleDown 5분 | 안정화 | 급하게 줄이지 않음 |
| scaleUp 즉시 | 빠른 대응 | 확장은 바로 |

#### cms-celery-worker (작업 처리)

```yaml
# kubernetes/base/hpa/hpa-celery-worker.yaml
spec:
  scaleTargetRef:
    name: cms-celery-worker

  minReplicas: 1
  maxReplicas: 3                # Worker는 3개까지

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          averageUtilization: 60   # CPU 60% (집약적 작업)
    - type: Resource
      resource:
        name: memory
        target:
          averageUtilization: 70   # Memory 70%

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600  # 10분 대기 (장시간 작업)
```

**Celery Worker 특성:**

| 특성 | 설정 이유 |
|------|----------|
| CPU 60% | 변환 작업이 CPU 집약적 |
| 축소 10분 | 장시간 작업 완료 대기 |
| 최대 3개 | 병렬 처리 한계 |

#### cms-web-ui (프론트엔드)

```yaml
# kubernetes/base/hpa/hpa-web-ui.yaml
spec:
  scaleTargetRef:
    name: cms-web-ui

  minReplicas: 1
  maxReplicas: 3

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          averageUtilization: 70
```

---

## 6. 실제 적용 방법

### 전제 조건

#### 1. Metrics Server 확인

```bash
# Metrics Server가 설치되어 있어야 HPA 동작
kubectl get deployment metrics-server -n kube-system

# Pod 메트릭 확인 가능한지 테스트
kubectl top pods -n dev
```

**Metrics Server 없으면:**
```bash
# AKS는 기본 설치됨, 없으면 설치
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### 2. Deployment에 resources 설정 필수

```yaml
# HPA가 CPU/Memory 비율 계산하려면 requests 필수!
resources:
  requests:        # ← 이게 있어야 HPA 동작
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "1Gi"
```

**현재 상태:** deployment-web-api.yaml에 이미 설정됨 ✅

### 적용 단계

#### Step 1: 현재 상태 확인

```bash
# 현재 HPA 있는지 확인
kubectl get hpa -n dev

# 결과 예시 (없으면)
No resources found in dev namespace.
```

#### Step 2: Kustomize로 배포

```bash
# HPA 포함 전체 배포
kubectl apply -k kubernetes/overlays/dev

# 또는 HPA만 개별 적용
kubectl apply -f kubernetes/base/hpa/hpa-web-api.yaml -n dev
kubectl apply -f kubernetes/base/hpa/hpa-celery-worker.yaml -n dev
kubectl apply -f kubernetes/base/hpa/hpa-web-ui.yaml -n dev
```

#### Step 3: 적용 확인

```bash
# HPA 목록 확인
kubectl get hpa -n dev

# 결과 예시
NAME                    REFERENCE                     TARGETS           MINPODS   MAXPODS   REPLICAS
cms-web-api-hpa         Deployment/cms-web-api        cpu: 25%/70%      1         5         1
cms-celery-worker-hpa   Deployment/cms-celery-worker  cpu: 10%/60%      1         3         1
cms-web-ui-hpa          Deployment/cms-web-ui         cpu: 15%/70%      1         3         1
```

**TARGETS 열 해석:**
- `cpu: 25%/70%` → 현재 25%, 목표 70%
- `<unknown>/70%` → Metrics Server 문제 또는 Pod 시작 전

#### Step 4: 상세 정보 확인

```bash
# HPA 상세 정보
kubectl describe hpa cms-web-api-hpa -n dev
```

**출력 예시:**
```
Name:                     cms-web-api-hpa
Namespace:                dev
Reference:                Deployment/cms-web-api
Metrics:
  "cpu" resource:
    current: 25%
    target:  70%
  "memory" resource:
    current: 45%
    target:  80%
Min replicas:             1
Max replicas:             5
Deployment pods:          1 current / 1 desired
Events:
  Type    Reason             Age   Message
  ----    ------             ----  -------
  Normal  SuccessfulRescale  10m   New size: 2; reason: cpu resource utilization above target
  Normal  SuccessfulRescale  5m    New size: 1; reason: All metrics below target
```

---

## 7. 동작 테스트

### 부하 테스트로 HPA 동작 확인

#### Step 1: 현재 상태 모니터링 (터미널 1)

```bash
# 실시간 HPA 상태 확인
kubectl get hpa -n dev -w
```

#### Step 2: 부하 생성 (터미널 2)

```bash
# 반복 요청으로 CPU 부하 생성
while true; do
  curl -s http://your-api-endpoint/health > /dev/null
done

# 또는 hey 도구 사용 (권장)
hey -n 10000 -c 100 http://your-api-endpoint/api/v1/some-endpoint
```

#### Step 3: 확장 확인

```
# 터미널 1 출력 변화
NAME              REFERENCE              TARGETS      REPLICAS
cms-web-api-hpa   Deployment/cms-web-api cpu: 25%/70%    1
cms-web-api-hpa   Deployment/cms-web-api cpu: 65%/70%    1
cms-web-api-hpa   Deployment/cms-web-api cpu: 82%/70%    1   # 70% 초과!
cms-web-api-hpa   Deployment/cms-web-api cpu: 82%/70%    2   # Pod 추가됨
cms-web-api-hpa   Deployment/cms-web-api cpu: 45%/70%    2   # 부하 분산
```

#### Step 4: 축소 확인

```bash
# 부하 중지 후 5분(stabilizationWindowSeconds) 대기
# Pod가 자동으로 줄어드는지 확인

kubectl get hpa -n dev -w
# REPLICAS: 2 → 1
```

---

## 8. 환경별 HPA 조정

### DEV vs PRD 설정 차이

현재 base HPA를 그대로 사용하지만, 환경별로 다르게 설정하려면:

#### DEV 환경 (리소스 절약)

```yaml
# overlays/dev/hpa-patch.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cms-web-api-hpa
spec:
  minReplicas: 1
  maxReplicas: 2    # DEV는 최대 2개로 제한
```

```yaml
# overlays/dev/kustomization.yaml에 추가
patches:
  - path: hpa-patch.yaml
```

#### PRD 환경 (안정성 우선)

```yaml
# overlays/prd/hpa-patch.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cms-web-api-hpa
spec:
  minReplicas: 2    # PRD는 최소 2개 유지
  maxReplicas: 10   # PRD는 최대 10개까지
```

### 권장 설정

| 환경 | 컴포넌트 | minReplicas | maxReplicas |
|------|---------|-------------|-------------|
| DEV | web-api | 1 | 2 |
| DEV | celery-worker | 1 | 2 |
| DEV | web-ui | 1 | 2 |
| PRD | web-api | 2 | 5 |
| PRD | celery-worker | 1 | 5 |
| PRD | web-ui | 2 | 3 |

---

## 9. 주의사항

### HPA + Deployment replicas 충돌

```yaml
# Deployment
spec:
  replicas: 1       # ← 이 값은 초기값일 뿐

# HPA
spec:
  minReplicas: 2    # ← HPA가 이 값으로 덮어씀
```

**결론:** HPA 적용 후에는 Deployment의 replicas 값은 무시됨

### HPA가 동작 안 할 때 체크리스트

| 증상 | 원인 | 해결 |
|------|------|------|
| TARGETS: `<unknown>` | Metrics Server 없음 | Metrics Server 설치 |
| TARGETS: `<unknown>` | resources.requests 없음 | Deployment에 requests 추가 |
| 확장 안 됨 | CPU가 목표값 미만 | 목표값 낮추기 또는 부하 확인 |
| 축소 안 됨 | stabilizationWindow 대기 중 | 설정된 시간 대기 |

### Stateful 앱 주의

```
Redis, PostgreSQL 등 Stateful 앱은 HPA 부적합!

이유:
- 데이터 동기화 문제
- 리더 선출 복잡성
- 스토리지 충돌

→ StatefulSet + 수동 스케일링 권장
```

---

## 10. 요약

### Before vs After

| 구분 | HPA 없음 | HPA 있음 |
|------|---------|---------|
| 트래픽 대응 | 수동 (5~10분) | 자동 (15초) |
| 장애 위험 | 높음 | 낮음 |
| 비용 | 고정 | 최적화 (30~50% 절감) |
| 운영 부담 | 높음 | 낮음 |
| 야간 리소스 | 낭비 | 자동 축소 |

### 기억할 것

```
1. HPA = Pod 개수 자동 조절
2. Metrics Server 필수 (AKS 기본 설치)
3. Deployment에 resources.requests 필수
4. 확장은 빠르게, 축소는 천천히 (안정화)
5. Stateful 앱은 HPA 부적합
```

### 적용 명령어 요약

```bash
# 1. 전제 조건 확인
kubectl top pods -n dev

# 2. 배포
kubectl apply -k kubernetes/overlays/dev

# 3. 확인
kubectl get hpa -n dev

# 4. 상세 확인
kubectl describe hpa cms-web-api-hpa -n dev

# 5. 실시간 모니터링
kubectl get hpa -n dev -w
```

---

## 11. 관련 문서

- [Kubernetes HPA 공식 문서](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [AKS 자동 크기 조정](https://learn.microsoft.com/ko-kr/azure/aks/concepts-scale)
- [환경변수 설정 현황](./환경변수-설정현황.md)
