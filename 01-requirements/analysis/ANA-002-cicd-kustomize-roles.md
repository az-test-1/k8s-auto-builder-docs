# CI/CD vs Kustomize 역할 분담

## 현재 상황 (복잡함)

```
┌─────────────────────────────────────────────────────────────┐
│                      CI/CD (GitHub Actions)                  │
├─────────────────────────────────────────────────────────────┤
│ 1. Key Vault에서 시크릿 가져옴                                │
│ 2. kubectl create secret 으로 K8s Secret 생성                │
│ 3. Docker 이미지 빌드/푸시                                    │
│ 4. kustomization.yaml의 images 태그 업데이트                  │
│ 5. kubectl apply -k 실행                                     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                      Kustomize                               │
├─────────────────────────────────────────────────────────────┤
│ 1. ConfigMap 패치 (환경별 값)                                 │
│ 2. images 섹션 (이미지 교체)  ← CI/CD에서 또 수정함            │
│ 3. Replica 수 조정                                           │
│ 4. PVC 크기 조정                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 문제점

| 문제 | 설명 |
|------|------|
| **중복** | 이미지 태그를 CI/CD에서도, Kustomize에서도 관리 |
| **혼란** | Secret은 CI/CD에서 생성, ConfigMap은 Kustomize에서 패치 |
| **일관성 없음** | ACR_REGISTRY가 ConfigMap에도, images에도 있음 |
| **파일 수정 충돌** | CI/CD가 kustomization.yaml 직접 수정 → Git 충돌 가능 |

---

## 올바른 기준

### CI/CD가 해야 할 것 (동적, 런타임)

| 항목 | 이유 |
|------|------|
| **시크릿 생성** | Key Vault에서 가져와서 주입 (보안) |
| **이미지 빌드/푸시** | 코드 변경 시마다 새 이미지 |
| **이미지 태그 결정** | 빌드 시점에 태그 생성 (날짜/커밋) |
| **배포 실행** | `kubectl apply` |

### Kustomize가 해야 할 것 (정적, 선언적)

| 항목 | 이유 |
|------|------|
| **환경별 설정** | dev/prd ConfigMap 차이 |
| **리소스 스펙** | Replica, CPU, Memory, PVC 크기 |
| **네임스페이스** | 환경별 분리 |
| **이미지 이름** | 레지스트리 주소 (태그 제외) |

---

## 현재 CI/CD 문제 코드

```yaml
# .github/workflows/ci-cd.yaml (line 293-321)
- name: Update Image Tags in Kustomization
  run: |
    cd kubernetes/overlays/${{ env.ENVIRONMENT }}

    # kustomization.yaml 파일을 직접 수정 ← 문제!
    if ! grep -q "images:" kustomization.yaml; then
      cat >> kustomization.yaml << EOF
      images:
        - name: ...
          newTag: "${{ needs.build.outputs.image_tag }}"
      EOF
    else
      yq -i '.images[0].newTag = "..."' kustomization.yaml
    fi
```

### 문제점

| 문제 | 현재 상황 |
|------|----------|
| **파일 수정 충돌** | CI/CD가 kustomization.yaml 직접 수정 → Git 충돌 가능 |
| **중복 관리** | images를 Kustomize에도, CI/CD에서도 설정 |
| **복잡한 로직** | `if ! grep -q "images:"` 같은 조건문 필요 |

---

## 권장 구조 (단순화)

### 방법 1: `kustomize edit set image` 사용

```bash
# CI/CD에서 (파일 수정 없이 메모리에서 처리)
cd kubernetes/overlays/$ENVIRONMENT
kustomize edit set image \
  acrdemo01061855.azurecr.io/auto-builder-cms-main=$SERVICE_ACR/$APP_NAME:$TAG

kubectl apply -k .
```

### 방법 2: `kubectl set image` 사용

```bash
# Kustomize 적용 후 이미지만 업데이트
kubectl apply -k kubernetes/overlays/$ENVIRONMENT

kubectl set image deployment/cms-web-api \
  cms-web-api=$SERVICE_ACR/$APP_NAME:$TAG -n $NAMESPACE

kubectl set image deployment/cms-web-mcp \
  cms-web-mcp=$SERVICE_ACR/$APP_NAME:$TAG -n $NAMESPACE

kubectl set image deployment/cms-celery-worker \
  celery-worker=$SERVICE_ACR/$APP_NAME:$TAG -n $NAMESPACE
```

### 방법 3: 환경변수로 태그 주입 (ArgoCD 스타일)

```yaml
# kustomization.yaml (정적)
images:
  - name: acrdemo01061855.azurecr.io/auto-builder-cms-main
    newName: acraz01cloudtr.azurecr.io/auto-builder-cms
    newTag: latest  # 기본값

# CI/CD에서 --set 으로 오버라이드
kustomize build . | kubectl apply -f -
# 또는
kubectl apply -k . --image=...
```

---

## 역할 분담 기준표

| 기준 | CI/CD | Kustomize |
|------|-------|-----------|
| **변경 주기** | 배포마다 | 환경 구성 변경 시 |
| **보안** | 시크릿 (Key Vault → K8s Secret) | 공개 설정만 |
| **이미지** | 태그 (동적) | 레지스트리/이름 (정적) |
| **저장** | 실행 시 생성 | Git에 저장 |
| **버전 관리** | 불필요 | Git으로 관리 |

---

## 현재 vs 권장

| 항목 | 현재 | 권장 | 비고 |
|------|------|------|------|
| Secret 생성 | CI/CD | CI/CD | 유지 |
| ConfigMap | Kustomize | Kustomize | 유지 |
| 이미지 레지스트리 | CI/CD + Kustomize | Kustomize만 | 중복 제거 |
| 이미지 태그 | CI/CD (파일 수정) | CI/CD (`kubectl set image`) | 단순화 |
| ACR_REGISTRY | ConfigMap + Kustomize | Kustomize만 | 중복 제거 |

---

## 개선 작업 목록

### 1단계: 중복 제거
- [ ] ConfigMap의 ACR_REGISTRY 제거 (앱에서 사용 안 하면)
- [ ] 또는 Kustomize images에서 ConfigMap 참조

### 2단계: CI/CD 단순화
- [ ] kustomization.yaml 파일 수정 로직 제거
- [ ] `kubectl set image` 방식으로 변경

### 3단계: 정리
- [ ] 문서 업데이트
- [ ] 테스트 배포

---

## 최종 권장 구조

```
┌─────────────────────────────────────────────────────────────┐
│  Git Repository (정적, 버전 관리)                            │
├─────────────────────────────────────────────────────────────┤
│  kubernetes/                                                │
│  ├── base/           ← 공통 리소스                          │
│  │   ├── cms/configmap.yaml    ← 공개 설정                  │
│  │   ├── cms/deployment-*.yaml ← 플레이스홀더 이미지         │
│  │   └── ...                                                │
│  └── overlays/                                              │
│      ├── dev/kustomization.yaml  ← 환경별 패치, 이미지 이름  │
│      └── prd/kustomization.yaml                             │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  CI/CD (동적, 배포 시)                                       │
├─────────────────────────────────────────────────────────────┤
│  1. Key Vault → K8s Secret 생성                             │
│  2. Docker 이미지 빌드/푸시 (태그: 날짜.시간)                  │
│  3. kubectl apply -k overlays/$ENV                          │
│  4. kubectl set image deployment/* $IMAGE:$TAG              │
└─────────────────────────────────────────────────────────────┘
```
