# ADR-M001: 컨테이너 오케스트레이션 플랫폼 선택

> **Status**: 승인됨
> **Date**: 2025-12
> **Category**: Migration - Infrastructure

---

## 상태
✅ **승인됨** (2025-12)

## 컨텍스트
기존 시스템은 단일 VM에서 Docker Compose로 운영되었으나, 다음과 같은 한계에 직면:
- 단일 장애점(SPOF) 존재
- 수동 스케일링만 가능
- 무중단 배포 불가
- 리소스 활용 비효율

## 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. Docker Compose 유지** | 단순함, 학습곡선 없음 | 확장성/가용성 한계 |
| **B. Docker Swarm** | Compose와 유사, 학습 용이 | 커뮤니티 축소, 기능 제한 |
| **C. Azure Kubernetes Service (AKS)** | 관리형, Azure 통합, 생태계 | 학습곡선, 복잡성 |
| **D. Azure Container Apps** | 서버리스, 간편함 | 커스터마이징 제한 |

## 결정
**옵션 C: Azure Kubernetes Service (AKS)** 선택

## 근거
1. **관리형 서비스**: Control Plane 관리 부담 제거
2. **Azure 네이티브 통합**: Key Vault, ACR, Azure Files 등과 원활한 연동
3. **확장성**: HPA, Cluster Autoscaler 지원
4. **생태계**: Helm, Kustomize 등 풍부한 도구 지원
5. **비용 효율**: Control Plane 무료, Node만 과금
6. **향후 확장성**: 복잡한 워크로드 대응 가능

## 결과
- AKS 클러스터 2개 생성 (DEV, PRD)
- Node Pool: Standard_D2s_v3 (2 vCPU, 8GB)
- Kubernetes 버전: 1.28+

## 관련 파일
```
azure/scripts/03-aks-cluster.sh
```

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
