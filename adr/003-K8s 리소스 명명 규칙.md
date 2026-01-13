# ADR-003: K8s 리소스 명명 규칙 표준화

## 상태
채택됨 (Accepted)

## 날짜
2026-01-12

## 컨텍스트

기존 K8s 리소스명이 일관성 없이 명명되어 있었다:

```
# 기존 명명 (비일관적)
cms-web-api, cms-web-mcp, cms-celery-worker
auto-builder-prd, auto-builder-dev
cms-config, cms-secrets
```

문제점:
1. `cms`가 `ui`와 대칭되지 않음 (CMS는 Content Management System의 약자로 백엔드 전체를 표현하기 부적절)
2. `autobuilder` 서비스명이 너무 길어 리소스명 조합 시 가독성 저하
3. 이미지 placeholder(`cms-image`)가 실제 용도(api, mcp, worker 공용)와 불일치
4. namespace명(`auto-builder-prd`)이 불필요하게 김

## 고려한 대안

### 백엔드 폴더/서비스 그룹명

| 옵션 | 장점 | 단점 |
|------|------|------|
| `api` | 직관적, 짧음 | api/mcp/worker 중 api만 지칭하는 것처럼 보임 |
| `backend` | ui와 대칭, 포괄적 | 다소 김 (7자) |
| `server` | 범용적 | 명확성 부족 |
| `core` | 핵심 서비스 강조 | 다소 추상적 |
| `svc` | 짧음 | 약어라 직관성 떨어짐 |

### 이미지 placeholder명

| 옵션 | 장점 | 단점 |
|------|------|------|
| `api-image` | 짧음 | worker, mcp도 같은 이미지인데 오해 소지 |
| `backend-image` | 정확한 표현 | 7자로 다소 김 |
| `server-image` | 범용적 | 모호함 |

### 데이터 경로

| 옵션 | 장점 | 단점 |
|------|------|------|
| `/data/cms/` 유지 | 기존 데이터 호환 | 명명 규칙 불일치 |
| `/data/ab/` | 서비스명과 일치 | 데이터 마이그레이션 필요 |
| `/data/backend/` | 폴더명과 일치 | 데이터 마이그레이션 필요 |

## 결정

### 1. 서비스 약어: `ab` (Auto-Builder)
```bash
# 00-env.sh
export SERVICE_NAME="ab"  # ab = Auto-Builder
```

### 2. 폴더 구조
```
k8s/
├── base/
│   ├── backend/     # cms → backend (ui와 대칭)
│   └── ui/
└── overlays/
    ├── dev/
    │   ├── backend/
    │   └── ui/
    └── prd/
        ├── backend/
        └── ui/
```

### 3. K8s 리소스 명명 규칙

| 리소스 유형 | 명명 패턴 | 예시 |
|------------|----------|------|
| Deployment | `ab-{역할}` | ab-api, ab-mcp, ab-worker, ab-ui |
| Service | `ab-{역할}` | ab-api, ab-mcp, ab-ui |
| ConfigMap | `ab-config` | ab-config |
| Secret | `ab-secrets` | ab-secrets |
| PVC | `ab-pvc` | ab-pvc |
| HPA | `ab-{역할}-hpa` | ab-api-hpa, ab-worker-hpa |
| PDB | `ab-{역할}-pdb` | ab-api-pdb, ab-worker-pdb |
| Ingress | `ab-ingress` | ab-ingress |
| ServiceAccount | `ab-sa` | ab-sa |
| StatefulSet | `ab-{용도}` | ab-redis |

### 4. Namespace
```yaml
# 환경별 단순 명명
dev   # (기존: auto-builder-dev)
prd   # (기존: auto-builder-prd)
```

### 5. Container Registry 이미지
```
acraz01ab01.azurecr.io/ab-backend-{env}   # backend (api, mcp, worker 공용)
acraz01ab01.azurecr.io/ab-ui-{env}        # frontend
```

### 6. 이미지 Placeholder
```yaml
# base/backend/deployment-*.yaml
image: backend-image   # (기존: cms-image)

# base/ui/deployment.yaml
image: ui-image
```

### 7. 데이터 경로
```yaml
# 기존 호환성 유지
mountPath: /data/cms    # 변경 없음
```
- 이유: 기존 PVC 데이터 마이그레이션 불필요
- 향후 신규 구축 시 `/data/ab/` 사용 권장

### 8. ArgoCD Application
```yaml
# argocd/apps/
backend.yaml      # (기존: cms.yaml)
backend-dev.yaml  # (기존: cms-dev.yaml)
ui.yaml
ui-dev.yaml
```

## 결과

### 장점
- `ui` ↔ `backend` 대칭으로 직관적인 구조
- 짧은 서비스명(`ab`)으로 리소스명 간결화
- 이미지명이 실제 용도와 일치 (`backend-image`가 api, mcp, worker 공용임을 명확히 표현)
- namespace 단순화로 kubectl 사용 편의성 향상

### 단점
- 기존 리소스 전체 재배포 필요
- ACR 이미지 재태그 또는 재빌드 필요
- CI/CD 파이프라인 이미지명 수정 필요

### 영향 범위

| 항목 | 수량 |
|------|------|
| 폴더명 변경 | 6개 |
| 파일명 변경 | 5개 |
| K8s 매니페스트 수정 | 14개 파일 |
| 스크립트 수정 | 2개 파일 |
| 문서 수정 | 3개 파일 |
| ACR 이미지 | 2개 |

## 참고
- Azure 리소스 명명 규칙: `[리소스약어]-az01-[환경]-ab-[용도]-[시퀀스]`
- Kubernetes 권장 라벨: https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/
