# CI/CD vs Kustomize 구성 진단 결과

> 진단 기준: `CICD-Kustomize-역할분담.md`
> 진단 대상: `환경변수-설정현황.md`
> 최종 업데이트: 2026-01-08

---

## 판정: 100% 적합 (24/24) ✅

> 이전 진단 (67%) → 개선 후 (100%)

---

## 1. CI/CD 역할 체크

| 항목 | 권장 담당 | 현재 담당 | 판정 | 비고 |
|------|----------|----------|:----:|------|
| Secret 생성 (Key Vault → K8s) | CI/CD | CI/CD | **O** | 정상 |
| 이미지 빌드/푸시 | CI/CD | CI/CD | **O** | 정상 |
| 이미지 태그 결정 | CI/CD | CI/CD | **O** | 정상 |
| 배포 실행 | CI/CD | CI/CD | **O** | 정상 |
| kustomization.yaml 파일 수정 | X (하면 안됨) | X (안함) | **O** | ✅ 개선됨 |

**결과: 5/5 (100%)**

---

## 2. Kustomize 역할 체크

| 항목 | 권장 담당 | 현재 담당 | 판정 | 비고 |
|------|----------|----------|:----:|------|
| ConfigMap 환경별 패치 | Kustomize | Kustomize | **O** | 정상 |
| 네임스페이스 분리 | Kustomize | Kustomize | **O** | 정상 |
| Replica 수 조정 | Kustomize | Kustomize | **O** | 정상 |
| PVC 크기 조정 | Kustomize | Kustomize | **O** | 정상 |
| 이미지 레지스트리 (정적) | Kustomize | Kustomize | **O** | 정상 |

**결과: 5/5 (100%)**

---

## 3. 중복/충돌 체크

| 항목 | 상태 | 판정 | 비고 |
|------|------|:----:|------|
| BUILD_IMAGE_REGISTRY | ConfigMap (동적 Job용) | **O** | 용도 분리됨 |
| 이미지 태그 | CI/CD (`kubectl set image`) | **O** | ✅ 개선됨 |
| SERVICE_ACR | GitHub vars (빌드/푸시용) | **O** | 용도 분리됨 |

**결과: 3/3 (100%)**

---

## 4. 설정 위치 적절성 체크

| 설정 | 현재 위치 | 권장 위치 | 판정 | 비고 |
|------|----------|----------|:----:|------|
| DATABASE_URL | Key Vault → Secret | Key Vault → Secret | **O** | 정상 |
| JWT_SECRET_KEY | Key Vault → Secret | Key Vault → Secret | **O** | 정상 |
| OPENAI_KEY_* | Key Vault → Secret | Key Vault → Secret | **O** | 정상 |
| REDIS_PASSWORD | Key Vault → Secret | Key Vault → Secret | **O** | 정상 |
| REDIS_HOST | ConfigMap | ConfigMap | **O** | 정상 |
| K8S_NAMESPACE | Kustomize patch (add) | Kustomize patch | **O** | ✅ 개선됨 |
| BUILD_IMAGE_REGISTRY | ConfigMap + patch | ConfigMap + patch | **O** | 용도 명확 |
| LOG_LEVEL | ConfigMap + patch | Kustomize patch | **O** | 정상 |
| 이미지 이름 | Kustomize | Kustomize | **O** | 정상 |
| 이미지 태그 | CI/CD (`kubectl set image`) | CI/CD 동적 | **O** | ✅ 개선됨 |
| 스토리지 경로 | ConfigMap | ConfigMap | **O** | 정상 |

**결과: 11/11 (100%)**

---

## 5. 점수 요약

| 구분 | 이전 | 현재 | 변화 |
|------|:----:|:----:|:----:|
| CI/CD 역할 | 80% | 100% | +20% |
| Kustomize 역할 | 80% | 100% | +20% |
| 중복/충돌 | 0% | 100% | +100% |
| 설정 위치 | 73% | 100% | +27% |
| **합계** | **67%** | **100%** | **+33%** |

---

## 6. 개선 완료 항목

| # | 문제 | 이전 상태 | 개선 내용 | 상태 |
|---|------|----------|----------|:----:|
| 1 | CI/CD가 kustomization.yaml 직접 수정 | `yq -i`로 파일 수정 | `kubectl set image` 방식으로 변경 | ✅ |
| 2 | BUILD_IMAGE_REGISTRY 중복 | ConfigMap + Kustomize images | 용도 분리 (동적 Job vs 정적 Deployment) | ✅ |
| 3 | 이미지 관리 중복 | Kustomize images + CI/CD 수정 | CI/CD는 `kubectl set image`로 태그만 | ✅ |
| 4 | K8S_NAMESPACE 중복 | ConfigMap base + patch | base에서 제거, patch에서 `add` | ✅ |
| 5 | SERVICE_ACR 중복 | GitHub vars + Kustomize | 용도 분리 (빌드용 vs 배포용) | ✅ |

---

## 7. 현재 구조

```
┌─────────────────────────────────────────────────────────────┐
│  CI/CD (동적)                                               │
├─────────────────────────────────────────────────────────────┤
│  ✅ Key Vault → K8s Secret 생성                              │
│  ✅ Docker 이미지 빌드/푸시                                   │
│  ✅ kubectl apply -k overlays/$ENV                          │
│  ✅ kubectl set image deployment/* $IMAGE:$TAG              │
│  ✅ kustomization.yaml 파일 수정 안 함                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Kustomize (정적)                                           │
├─────────────────────────────────────────────────────────────┤
│  ✅ 환경별 ConfigMap 패치                                    │
│  ✅ 네임스페이스 분리                                        │
│  ✅ 리소스 스펙 (Replica, CPU, Memory, PVC)                  │
│  ✅ 이미지 레지스트리 이름 (태그는 latest)                    │
│  ✅ K8S_NAMESPACE는 overlay에서 add                          │
│  ✅ BUILD_IMAGE_REGISTRY는 overlay에서 patch                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. 변수 관리 표준 (최종)

| 변수 | 위치 | 용도 | 관리 방식 |
|------|------|------|----------|
| `SERVICE_ACR_LOGIN_SERVER` | GitHub vars | CI/CD 이미지 빌드/푸시 | Organization 변수 |
| `BUILD_IMAGE_REGISTRY` | ConfigMap + patch | K8s Job 동적 이미지 | Kustomize overlay |
| `K8S_NAMESPACE` | Kustomize patch (add) | 앱이 Job 생성 시 참조 | overlay에서만 정의 |
| 이미지 이름 | Kustomize images | Deployment 이미지 | overlay 정적 설정 |
| 이미지 태그 | CI/CD | Deployment 이미지 태그 | `kubectl set image` |

---

## 9. 배포 흐름 (최종)

```
1. CI/CD: 이미지 빌드 & 푸시
   └─ $SERVICE_ACR_LOGIN_SERVER/$APP_NAME:$TAG

2. CI/CD: Key Vault → K8s Secret 생성
   └─ db-credentials, app-secrets, acr-secret

3. CI/CD: Kustomize 적용 (정적 매니페스트)
   └─ kubectl apply -k kubernetes/overlays/$ENV

4. CI/CD: 이미지 태그 동적 업데이트
   └─ kubectl set image deployment/* $IMAGE:$TAG

5. CI/CD: 롤아웃 대기
   └─ kubectl rollout status deployment/*
```

---

## 10. 관련 문서

- [Container Registry 관리 표준](./ACR_REGISTRY-관리-표준.md)
- [CI/CD-Kustomize 역할분담](./CICD-Kustomize-역할분담.md)
- [환경변수 설정 현황](./환경변수-설정현황.md)
- [유즈케이스 Java8to17 실행흐름](./유즈케이스-Java8to17-실행흐름.md)
