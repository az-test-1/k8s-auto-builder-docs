# ADR-M003: Kubernetes 네임스페이스 전략

> **Status**: 승인됨
> **Date**: 2026-01
> **Category**: Migration - Kubernetes

---

## 상태
✅ **승인됨** (2026-01)

## 컨텍스트
환경(DEV/PRD)별 네임스페이스 명명 규칙 결정 필요

## 고려한 옵션

| 옵션 | 예시 | 장점 | 단점 |
|------|------|------|------|
| **A. 서비스명 포함** | `auto-builder-dev` | 명시적 | 길고 중복적 |
| **B. 환경명만** | `dev`, `prd` | 간결함 | 다른 서비스와 충돌 가능 |
| **C. 팀/프로젝트 접두사** | `cloudtr-dev` | 팀 구분 | 적당히 복잡 |

## 결정
**옵션 B: 환경명만 사용** (`dev`, `prd`)

## 근거
1. **클러스터 분리**: DEV/PRD가 별도 클러스터이므로 충돌 위험 없음
2. **간결성**: kubectl 명령어 타이핑 간소화
3. **표준화**: 많은 조직에서 사용하는 관행
4. **Kustomize 호환**: overlay 구조와 자연스럽게 매칭

## 결과
```yaml
# kubernetes/overlays/dev/kustomization.yaml
namespace: dev

# kubernetes/overlays/prd/kustomization.yaml
namespace: prd
```

## 변경된 파일
- `kubernetes/overlays/dev/kustomization.yaml`
- `kubernetes/overlays/prd/kustomization.yaml`
- 모든 문서에서 `auto-builder-dev` → `dev` 변경

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
