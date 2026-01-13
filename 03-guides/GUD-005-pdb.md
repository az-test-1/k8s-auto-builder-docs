# [학습] PDB (PodDisruptionBudget)

> 노드 점검/업그레이드 중에도 서비스가 죽지 않게 보호하는 설정

---

## 1. PDB란?

### 한 줄 정의

**"노드 유지보수 시 최소 몇 개의 Pod는 살려둬야 한다"를 정의하는 정책**

### 시각적 이해

```
┌────────────────────────────────────────────────────────────┐
│  상황: AKS 노드 업그레이드로 Pod를 다른 노드로 이동해야 함    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  PDB 없음:                                                 │
│                                                            │
│  Node A                        Node B                      │
│  ┌─────┐ ┌─────┐              (비어있음)                   │
│  │Pod 1│ │Pod 2│  ─────────────────────►  전부 동시 종료   │
│  └─────┘ └─────┘                          서비스 다운 ❌   │
│                                                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  PDB 있음 (minAvailable: 1):                               │
│                                                            │
│  Node A                        Node B                      │
│  ┌─────┐ ┌─────┐              ┌─────┐                      │
│  │Pod 1│ │Pod 2│  ──Step1──►  │Pod 3│  새 Pod 먼저 생성    │
│  └─────┘ └─────┘              └─────┘                      │
│     │                            ✓                         │
│     └──────Step2──────────────────────►  그 다음 기존 종료  │
│                                          서비스 유지 ✅     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 2. 왜 필요해?

### PDB가 없으면 발생하는 문제

#### 시나리오 1: AKS 노드 업그레이드

```
Azure: "AKS 1.28 → 1.29 업그레이드 합니다"
      ↓
노드 하나씩 drain (Pod 퇴거)
      ↓
┌─────────────────────────────────────────┐
│  Node A (업그레이드 대상)                │
│  ┌──────────┐ ┌──────────┐              │
│  │ web-api  │ │ web-api  │  ← 둘 다 종료 │
│  │  Pod 1   │ │  Pod 2   │              │
│  └──────────┘ └──────────┘              │
└─────────────────────────────────────────┘
      ↓
모든 API Pod 종료 → 서비스 다운 5~10분 ❌
```

#### 시나리오 2: 노드 자동 스케일링 (축소)

```
AKS Autoscaler: "야간이라 노드 3개 → 2개로 줄일게"
      ↓
Node C의 모든 Pod 강제 퇴거
      ↓
┌─────────────────────────────────────────┐
│  Node C (제거 대상)                      │
│  ┌──────────┐ ┌──────────┐              │
│  │ celery   │ │  redis   │  ← 작업 중단! │
│  │ worker   │ │          │              │
│  └──────────┘ └──────────┘              │
└─────────────────────────────────────────┘
      ↓
진행 중인 변환 작업 실패 ❌
```

#### 시나리오 3: 스팟 인스턴스 (비용 절감 노드)

```
Azure: "스팟 인스턴스 회수합니다 (2분 내 종료)"
      ↓
해당 노드의 모든 Pod 강제 종료
      ↓
대응 시간 부족 → 서비스 장애 ❌
```

### PDB가 있으면

```
Kubernetes: "Node A drain 해야 하는데, Pod 종료해도 될까?"
      ↓
PDB Controller: "잠깐! cms-web-api는 최소 1개 유지해야 해"
      ↓
Kubernetes: "알겠어, 먼저 다른 노드에 새 Pod 띄울게"
      ↓
새 Pod Ready 확인 후 → 기존 Pod 종료
      ↓
서비스 중단 없음 ✅
```

---

## 3. 핵심 개념

### Voluntary vs Involuntary Disruption

| 구분 | Voluntary (자발적) | Involuntary (비자발적) |
|------|-------------------|----------------------|
| 원인 | 관리자/시스템 의도 | 예상치 못한 장애 |
| 예시 | 노드 drain, 업그레이드, 스케일링 | 하드웨어 장애, OOM Kill |
| PDB 적용 | **적용됨** ✅ | 적용 안 됨 ❌ |

**PDB는 "계획된" 중단만 보호한다!**

```
✅ PDB가 막아주는 것:
   - kubectl drain
   - 노드 업그레이드
   - 클러스터 오토스케일러 (노드 축소)
   - kubectl delete pod (eviction API)

❌ PDB가 못 막는 것:
   - 노드 갑자기 죽음
   - Pod OOM (메모리 부족)
   - 앱 크래시
   - 네트워크 장애
```

### 핵심 설정 2가지

#### 1. minAvailable (최소 유지)

```yaml
spec:
  minAvailable: 1    # 항상 1개 이상 유지
```

```
Pod 3개 있을 때:
├─ 동시에 2개까지 종료 가능 (3 - 1 = 2)
└─ 1개는 반드시 남아있어야 함
```

#### 2. maxUnavailable (최대 중단)

```yaml
spec:
  maxUnavailable: 1  # 최대 1개만 동시 중단 가능
```

```
Pod 3개 있을 때:
├─ 동시에 1개까지만 종료 가능
└─ 2개는 반드시 남아있어야 함
```

#### 어떤 걸 써야 해?

| 상황 | 권장 설정 |
|------|----------|
| 최소 가용성 보장 | `minAvailable: 1` |
| 롤링 업데이트 속도 조절 | `maxUnavailable: 1` |
| Pod 개수가 유동적 (HPA) | `minAvailable` 권장 |
| Pod 개수가 고정 | 둘 다 OK |

---

## 4. 현재 프로젝트 PDB 설정

### 파일 구조

```
kubernetes/
└── base/
    └── pdb/
        ├── kustomization.yaml
        ├── pdb-web-api.yaml        # API 서버 보호
        ├── pdb-celery-worker.yaml  # Worker 보호
        └── pdb-redis.yaml          # Redis 보호
```

### 각 PDB 설정

#### cms-web-api-pdb

```yaml
# kubernetes/base/pdb/pdb-web-api.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cms-web-api-pdb
spec:
  minAvailable: 1              # 최소 1개 유지
  selector:
    matchLabels:
      app: cms-web-api         # 이 라벨 가진 Pod에 적용
```

**의미:**
- API 서버 Pod가 여러 개 있어도 최소 1개는 항상 유지
- 노드 업그레이드 시에도 API 응답 가능

#### cms-celery-worker-pdb

```yaml
# kubernetes/base/pdb/pdb-celery-worker.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cms-celery-worker-pdb
spec:
  minAvailable: 1              # 최소 1개 유지
  selector:
    matchLabels:
      app: cms-celery-worker
```

**의미:**
- 변환 작업 처리 중에도 Worker 1개는 유지
- 진행 중인 작업이 갑자기 끊기지 않음

#### redis-pdb

```yaml
# kubernetes/base/pdb/pdb-redis.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: redis-pdb
spec:
  minAvailable: 1              # 최소 1개 유지
  selector:
    matchLabels:
      app: redis
```

**의미:**
- Redis는 단일 인스턴스라 반드시 1개 유지
- Redis 죽으면 Celery 전체 장애

---

## 5. 동작 예시

### 시나리오: AKS 노드 업그레이드

```
초기 상태:
┌─────────────────────────────────────────────────────────────┐
│  Node A                          Node B                     │
│  ┌───────────┐ ┌───────────┐    ┌───────────┐              │
│  │ web-api-1 │ │ celery-1  │    │ web-api-2 │              │
│  └───────────┘ └───────────┘    └───────────┘              │
│                                                             │
│  PDB 설정:                                                  │
│  - cms-web-api-pdb: minAvailable=1                         │
│  - cms-celery-worker-pdb: minAvailable=1                   │
└─────────────────────────────────────────────────────────────┘

Step 1: Node A drain 시작
┌─────────────────────────────────────────────────────────────┐
│  Kubernetes: "Node A 비워야 해"                             │
│                                                             │
│  PDB 체크:                                                  │
│  - web-api: 현재 2개, 1개 종료해도 1개 남음 → OK ✅          │
│  - celery: 현재 1개, 종료하면 0개 → BLOCKED ❌              │
└─────────────────────────────────────────────────────────────┘

Step 2: celery-2를 Node B에 먼저 생성
┌─────────────────────────────────────────────────────────────┐
│  Node A                          Node B                     │
│  ┌───────────┐ ┌───────────┐    ┌───────────┐ ┌───────────┐│
│  │ web-api-1 │ │ celery-1  │    │ web-api-2 │ │ celery-2  ││
│  └───────────┘ └───────────┘    └───────────┘ └───────────┘│
│                                                 (새로 생성) │
└─────────────────────────────────────────────────────────────┘

Step 3: celery-2 Ready 확인 후 celery-1 종료
┌─────────────────────────────────────────────────────────────┐
│  Node A                          Node B                     │
│  ┌───────────┐                  ┌───────────┐ ┌───────────┐│
│  │ web-api-1 │                  │ web-api-2 │ │ celery-2  ││
│  └───────────┘                  └───────────┘ └───────────┘│
│                                                             │
│  celery-1 종료됨, 하지만 celery-2가 이미 Ready ✅            │
└─────────────────────────────────────────────────────────────┘

Step 4: web-api-1 종료 (web-api-2가 있으므로 OK)
┌─────────────────────────────────────────────────────────────┐
│  Node A (비워짐)                 Node B                     │
│                                 ┌───────────┐ ┌───────────┐│
│                                 │ web-api-2 │ │ celery-2  ││
│                                 └───────────┘ └───────────┘│
└─────────────────────────────────────────────────────────────┘

Step 5: Node A 업그레이드 완료, 새 Pod들 다시 분배
┌─────────────────────────────────────────────────────────────┐
│  Node A (업그레이드됨)           Node B                     │
│  ┌───────────┐                  ┌───────────┐ ┌───────────┐│
│  │ web-api-3 │                  │ web-api-2 │ │ celery-2  ││
│  └───────────┘                  └───────────┘ └───────────┘│
│                                                             │
│  서비스 중단 시간: 0초 ✅                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 실제 적용 방법

### Step 1: 현재 상태 확인

```bash
# PDB 목록 확인
kubectl get pdb -n dev

# 결과 예시 (없으면)
No resources found in dev namespace.
```

### Step 2: Kustomize로 배포

```bash
# PDB 포함 전체 배포 (base에 이미 포함됨)
kubectl apply -k kubernetes/overlays/dev

# 또는 PDB만 개별 적용
kubectl apply -f kubernetes/base/pdb/ -n dev
```

### Step 3: 적용 확인

```bash
# PDB 목록 확인
kubectl get pdb -n dev

# 결과 예시
NAME                    MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
cms-web-api-pdb         1               N/A               1                     5m
cms-celery-worker-pdb   1               N/A               0                     5m
redis-pdb               1               N/A               0                     5m
```

**열 해석:**
| 열 | 의미 |
|-----|------|
| MIN AVAILABLE | 최소 유지해야 할 Pod 수 |
| ALLOWED DISRUPTIONS | 현재 종료 가능한 Pod 수 |

```
ALLOWED DISRUPTIONS = 현재 Pod 수 - minAvailable

예시:
- web-api Pod 2개, minAvailable 1 → ALLOWED: 1 (1개 종료 가능)
- celery Pod 1개, minAvailable 1 → ALLOWED: 0 (종료 불가)
```

### Step 4: 상세 정보 확인

```bash
kubectl describe pdb cms-web-api-pdb -n dev
```

**출력 예시:**
```
Name:           cms-web-api-pdb
Namespace:      dev
Min available:  1
Selector:       app=cms-web-api
Status:
    Allowed disruptions:  1
    Current:              2
    Desired:              1
    Total:                2
Events:         <none>
```

---

## 7. PDB 동작 테스트

### 테스트 1: drain 시 PDB 동작 확인

```bash
# 특정 노드 drain 시도
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# PDB가 blocking하면 아래 메시지 출력
error: cannot evict pod "cms-celery-worker-xxx",
  the disruption budget cms-celery-worker-pdb needs 1 healthy pods
```

### 테스트 2: 강제 eviction 시도

```bash
# Pod eviction 시도
kubectl delete pod cms-celery-worker-xxx -n dev

# 일반 delete는 PDB 무시하고 바로 삭제됨 (주의!)
# eviction API를 통해야 PDB 적용됨
```

**중요:** `kubectl delete pod`는 PDB를 우회함. drain이나 eviction API만 PDB 적용.

---

## 8. HPA + PDB 함께 사용

### 권장 조합

```yaml
# HPA 설정
spec:
  minReplicas: 1
  maxReplicas: 5

# PDB 설정 (HPA와 함께)
spec:
  minAvailable: 1    # ← 숫자로 지정 (퍼센트 아님)
```

### 주의사항

```
HPA minReplicas = 1
PDB minAvailable = 1

이 경우:
├─ Pod 1개일 때 → ALLOWED DISRUPTIONS: 0
├─ drain 불가 → 업그레이드 막힘!
└─ 해결: 먼저 Pod 늘리거나, 일시적으로 PDB 삭제
```

**권장 설정:**

| HPA minReplicas | PDB minAvailable | 결과 |
|-----------------|------------------|------|
| 1 | 1 | drain 시 일시적 지연 (새 Pod 먼저 생성) |
| 2 | 1 | drain 자유로움 ✅ |
| 3 | 2 | 고가용성 ✅ |

---

## 9. 환경별 PDB 조정

### DEV 환경 (단순)

```yaml
# DEV는 minAvailable: 1로 충분
spec:
  minAvailable: 1
```

### PRD 환경 (고가용성)

```yaml
# PRD는 더 높은 가용성 필요
# overlays/prd/pdb-patch.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cms-web-api-pdb
spec:
  minAvailable: 2    # PRD는 최소 2개 유지
```

```yaml
# overlays/prd/kustomization.yaml
patches:
  - path: pdb-patch.yaml
```

### 권장 설정 표

| 환경 | 컴포넌트 | minAvailable | 이유 |
|------|---------|--------------|------|
| DEV | web-api | 1 | 테스트 환경, 다운타임 허용 |
| DEV | celery | 1 | 작업 중단 최소화 |
| PRD | web-api | 2 | 고가용성, 무중단 |
| PRD | celery | 1 | 작업 연속성 |
| PRD | redis | 1 | 단일 인스턴스 |

---

## 10. 트러블슈팅

### 문제 1: drain이 멈춤

```bash
# 증상
kubectl drain node-1 --ignore-daemonsets
# ... 멈춤 (진행 안 됨)

# 원인 확인
kubectl get pdb -A

# 해결: ALLOWED DISRUPTIONS가 0인 PDB 확인
NAME              MIN AVAILABLE   ALLOWED DISRUPTIONS
redis-pdb         1               0                    # ← 이게 원인
```

**해결 방법:**
```bash
# 방법 1: 해당 앱의 Pod 수 먼저 늘리기
kubectl scale deployment redis --replicas=2

# 방법 2: 일시적으로 PDB 삭제 (위험!)
kubectl delete pdb redis-pdb -n dev
# drain 완료 후 다시 생성
kubectl apply -f kubernetes/base/pdb/pdb-redis.yaml
```

### 문제 2: PDB가 적용 안 됨

```bash
# 증상: PDB 있는데 Pod가 바로 종료됨

# 원인 1: selector가 안 맞음
kubectl get pdb cms-web-api-pdb -o yaml
# selector.matchLabels 확인

kubectl get pods --show-labels
# Pod label 확인

# 원인 2: kubectl delete는 PDB 무시
# → drain이나 eviction API 사용해야 PDB 적용
```

### 문제 3: PDB + HPA 충돌

```bash
# 증상: HPA가 Pod 줄이려는데 PDB가 막음

# 확인
kubectl describe hpa cms-web-api-hpa
# Events: "unable to scale down, pdb prevents"

# 해결: PDB minAvailable < HPA minReplicas 확인
```

---

## 11. 요약

### PDB vs HPA 비교

| 구분 | HPA | PDB |
|------|-----|-----|
| 목적 | 부하 대응 (확장) | 유지보수 보호 (안정성) |
| 언제 동작 | CPU/메모리 사용량 변화 | 노드 drain/업그레이드 |
| 뭘 하나 | Pod 개수 조절 | Pod 종료 제한 |
| 없으면 | 트래픽 급증 시 장애 | 점검 시 서비스 다운 |

### 핵심 정리

```
1. PDB = 노드 점검 시 Pod 보호
2. Voluntary disruption만 보호 (계획된 중단)
3. minAvailable: 1 → 최소 1개 유지
4. kubectl delete는 PDB 우회함 (drain 사용)
5. HPA와 함께 사용 시 minReplicas > minAvailable 권장
```

### 적용 명령어 요약

```bash
# 1. 배포 (base에 이미 포함)
kubectl apply -k kubernetes/overlays/dev

# 2. 확인
kubectl get pdb -n dev

# 3. 상세 확인
kubectl describe pdb cms-web-api-pdb -n dev

# 4. drain 테스트
kubectl drain <node-name> --ignore-daemonsets --dry-run=client
```

---

## 12. 관련 문서

- [Kubernetes PDB 공식 문서](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [AKS 노드 업그레이드](https://learn.microsoft.com/ko-kr/azure/aks/upgrade-cluster)
- [[학습] HPA](./[학습]%20HPA.md)
- [환경변수 설정 현황](./환경변수-설정현황.md)
