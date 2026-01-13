# ADR-M012: Redis 배포 방식

> **Status**: 승인됨
> **Date**: 2025-12
> **Category**: Migration - Cache

---

## 상태
✅ **승인됨** (2025-12)

## 컨텍스트
Celery 메시지 브로커 및 캐시로 사용되는 Redis 배포 방식 결정 필요

## 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. Azure Cache for Redis** | 관리형, HA 지원 | 비용 높음 |
| **B. K8s Deployment** | 단순 | 데이터 손실 위험 |
| **C. K8s StatefulSet** | 안정적 ID, PVC 지원 | 직접 운영 |
| **D. Redis Cluster (Helm)** | HA, 분산 | 복잡성, 과도한 스펙 |

## 결정
**옵션 C: Kubernetes StatefulSet**

## 근거
1. **비용 효율**: Azure Cache for Redis 대비 저렴
2. **현재 요구사항 충분**: 단일 인스턴스로 충분한 부하
3. **데이터 영속성**: PVC로 데이터 보존 가능
4. **단순성**: Redis Cluster까지 필요 없음

## 설정
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 1
  template:
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          args: ["--requirepass", "$(REDIS_PASSWORD)"]
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: managed-premium
        resources:
          requests:
            storage: 1Gi
```

## 향후 고려사항
- 트래픽 증가 시 Azure Cache for Redis로 마이그레이션 검토
- Redis Sentinel 또는 Cluster 구성 검토

## 관련 파일
```
kubernetes/base/redis/statefulset.yaml
kubernetes/base/redis/service.yaml
```

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
