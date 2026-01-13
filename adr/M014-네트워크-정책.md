# ADR-M014: 네트워크 정책

> **Status**: 보류
> **Date**: 2026-01
> **Category**: Migration - Security

---

## 상태
🔄 **보류** (향후 검토)

## 컨텍스트
Pod 간 네트워크 통신 제어를 위한 NetworkPolicy 적용 여부

## 현재 상태
- NetworkPolicy 미적용
- 모든 Pod 간 통신 허용

## 향후 검토 사항
```yaml
# 예시: API 서버만 DB 접근 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels:
      app: cms-web-api
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: <PostgreSQL_IP>/32
      ports:
        - protocol: TCP
          port: 5432
```

## 보류 사유
- 현재 AKS 내부 통신만 존재
- 외부 노출은 Ingress를 통해서만 가능
- 추가 복잡성 대비 보안 이점 낮음

## 적용 검토 시점
- 멀티 테넌트 환경 구성 시
- 규제 준수 요구사항 발생 시
- 외부 서비스 연동 증가 시

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
