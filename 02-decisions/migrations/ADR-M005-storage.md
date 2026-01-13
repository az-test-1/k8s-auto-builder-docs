# ADR-M005: 스토리지 솔루션

> **Status**: 승인됨
> **Date**: 2025-12
> **Category**: Migration - Storage

---

## 상태
✅ **승인됨** (2025-12)

## 컨텍스트
Auto-Builder는 프로젝트 소스 코드를 저장하고, 여러 Pod에서 동시에 접근해야 함. 적절한 K8s 스토리지 솔루션 선택 필요

## 고려한 옵션

| 옵션 | 접근 모드 | 성능 | 비용 |
|------|----------|------|------|
| **A. Azure Disk (Premium SSD)** | RWO | 높음 | 중간 |
| **B. Azure Files (Standard)** | RWX | 중간 | 낮음 |
| **C. Azure Files (Premium)** | RWX | 높음 | 높음 |
| **D. Azure NetApp Files** | RWX | 매우 높음 | 매우 높음 |
| **E. Azure Blob (NFS)** | RWX | 중간 | 낮음 |

## 결정
**옵션 C: Azure Files Premium**

## 근거
1. **RWX 지원**: 여러 Pod에서 동시 읽기/쓰기 필수 (빌드 Job + API 서버)
2. **SMB/NFS 지원**: K8s CSI 드라이버와 호환
3. **성능**: 빌드 작업 시 많은 파일 I/O 발생, Premium 성능 필요
4. **관리 용이**: Azure Portal에서 쉽게 관리

## 결과
```
DEV: stcloudtrdev (FileStorage, Premium_LRS)
PRD: stcloudtrprd (FileStorage, Premium_LRS)
File Share: cms-data (100 GiB)
```

## Kubernetes 설정
```yaml
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cms-data-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-csi-premium
  resources:
    requests:
      storage: 100Gi
```

## 관련 파일
```
azure/scripts/05-storage.sh
kubernetes/base/storage/pvc.yaml
```

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
