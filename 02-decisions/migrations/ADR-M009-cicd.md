# ADR-M009: CI/CD 파이프라인 및 배포 도구

> **Status**: 승인됨
> **Date**: 2025-12
> **Category**: Migration - CI/CD

---

## 상태
✅ **승인됨** (2025-12)

## 컨텍스트
K8s 환경에 맞는 CI/CD 파이프라인 및 매니페스트 관리 도구 선택 필요

## 고려한 옵션 (CI/CD)

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. GitHub Actions** | GitHub 통합, 무료 tier | Self-hosted runner 필요 시 비용 |
| **B. Azure DevOps Pipelines** | Azure 통합 | 별도 플랫폼 관리 |
| **C. GitLab CI** | 올인원 | GitHub에서 이전 필요 |
| **D. ArgoCD (GitOps)** | 선언적, 자동 동기화 | 추가 인프라 |

## 고려한 옵션 (매니페스트 관리)

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. 순수 YAML** | 단순 | 중복, 환경별 관리 어려움 |
| **B. Helm** | 템플릿, 패키지 관리 | 학습곡선, 복잡성 |
| **C. Kustomize** | 오버레이, kubectl 내장 | 템플릿 미지원 |
| **D. Helm + Kustomize** | 두 장점 결합 | 복잡성 증가 |

## 결정
- **CI/CD**: **GitHub Actions**
- **매니페스트 관리**: **Kustomize**

## 근거
1. **GitHub Actions**: 이미 GitHub 사용 중, 추가 도구 불필요
2. **Kustomize**:
   - kubectl 내장으로 추가 설치 불필요
   - base/overlay 구조로 환경별 설정 깔끔하게 관리
   - Helm 대비 학습곡선 낮음
   - 현재 요구사항에 충분

## 디렉토리 구조
```
kubernetes/
├── base/                 # 공통 리소스
│   ├── kustomization.yaml
│   ├── cms/
│   ├── ui/
│   ├── redis/
│   ├── storage/
│   ├── secrets/
│   └── ingress/
└── overlays/
    ├── dev/              # DEV 환경 오버라이드
    │   └── kustomization.yaml
    └── prd/              # PRD 환경 오버라이드
        └── kustomization.yaml
```

## CI/CD 흐름
```
1. Push to dev/main branch
2. GitHub Actions 트리거
3. Docker 이미지 빌드 & ACR Push
4. Key Vault에서 시크릿 조회
5. K8s Secret 생성
6. kustomize edit set image (태그 업데이트)
7. kubectl apply -k overlays/$ENV
8. kubectl rollout status (배포 확인)
```

## 관련 파일
```
.github/workflows/ci-cd.yaml
kubernetes/base/kustomization.yaml
kubernetes/overlays/*/kustomization.yaml
```

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
