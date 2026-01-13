# ADR-M007: Ingress Controller 선택

> **Status**: 승인됨
> **Date**: 2025-12
> **Category**: Migration - Networking

---

## 상태
✅ **승인됨** (2025-12)

## 컨텍스트
외부 트래픽을 K8s 서비스로 라우팅하기 위한 Ingress Controller 선택 필요

## 고려한 옵션

| 옵션 | 기능 | 복잡도 | Azure 통합 |
|------|------|:------:|:----------:|
| **A. NGINX Ingress Controller** | 표준, 다양한 기능 | 낮음 | 중간 |
| **B. Azure Application Gateway Ingress** | WAF, Azure 네이티브 | 중간 | 높음 |
| **C. Traefik** | 자동 설정, 미들웨어 | 중간 | 낮음 |
| **D. Contour (Envoy)** | 고성능, gRPC | 높음 | 낮음 |

## 결정
**옵션 A: NGINX Ingress Controller**

## 근거
1. **업계 표준**: 가장 널리 사용되어 문서/커뮤니티 풍부
2. **단순성**: 설치/설정이 간단
3. **기능 충분**: 현재 요구사항(경로 기반 라우팅)에 충분
4. **비용**: Application Gateway 대비 저렴
5. **이식성**: 다른 클라우드로 이전 시에도 동일하게 사용 가능

## 설치 방법
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

## Ingress 라우팅 규칙
```yaml
rules:
  - http:
      paths:
        - path: /api/auth
          pathType: Prefix
          backend:
            service:
              name: cms-web-ui    # NextAuth 처리
              port:
                number: 15001
        - path: /api
          pathType: Prefix
          backend:
            service:
              name: cms-web-api
              port:
                number: 18001
        - path: /mcp
          pathType: Prefix
          backend:
            service:
              name: cms-web-mcp
              port:
                number: 18002
        - path: /
          pathType: Prefix
          backend:
            service:
              name: cms-web-ui
              port:
                number: 15001
```

## 관련 파일
```
azure/scripts/06-ingress-controller.sh
kubernetes/base/ingress/ingress.yaml
```

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
