# [학습] Ingress

> 외부에서 들어오는 HTTP 요청을 내부 서비스로 라우팅하는 관문

---

## 1. Ingress란?

### 한 줄 정의

**"외부 트래픽을 URL 경로/도메인에 따라 내부 서비스로 분배하는 L7 로드밸런서"**

### 시각적 이해

```
┌─────────────────────────────────────────────────────────────────┐
│  인터넷 (외부)                                                   │
│                                                                 │
│  사용자 ──► https://myapp.com/         ──┐                      │
│  사용자 ──► https://myapp.com/api      ──┼──► Ingress           │
│  사용자 ──► https://myapp.com/api/auth ──┤      │               │
│  사용자 ──► https://myapp.com/mcp      ──┘      │               │
│                                                 │               │
├─────────────────────────────────────────────────┼───────────────┤
│  Kubernetes 클러스터 (내부)                      │               │
│                                                 ▼               │
│                                         ┌─────────────┐         │
│                                         │   Ingress   │         │
│                                         │  Controller │         │
│                                         └──────┬──────┘         │
│                          ┌──────────┬──────────┼──────────┐     │
│                          ▼          ▼          ▼          ▼     │
│                     ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ │
│                     │web-ui  │ │web-api │ │web-ui  │ │web-mcp │ │
│                     │Service │ │Service │ │Service │ │Service │ │
│                     └────────┘ └────────┘ └────────┘ └────────┘ │
│                        /         /api      /api/auth    /mcp    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 왜 필요해?

### Ingress 없이 외부 접근하는 방법들

#### 방법 1: NodePort

```
┌──────────────────────────────────────────────────────────────┐
│  각 서비스마다 포트 노출                                      │
│                                                              │
│  http://20.249.205.59:30001  ──►  web-ui                     │
│  http://20.249.205.59:30002  ──►  web-api                    │
│  http://20.249.205.59:30003  ──►  web-mcp                    │
└──────────────────────────────────────────────────────────────┘

문제점:
❌ 포트 번호 외우기 어려움 (30001? 30002?)
❌ 포트 범위 제한 (30000~32767)
❌ 보안 취약 (포트 직접 노출)
❌ HTTPS 적용 어려움
```

#### 방법 2: LoadBalancer

```
┌──────────────────────────────────────────────────────────────┐
│  각 서비스마다 공인 IP 할당                                   │
│                                                              │
│  http://20.249.205.59  ──►  web-ui                           │
│  http://20.249.205.60  ──►  web-api                          │
│  http://20.249.205.61  ──►  web-mcp                          │
└──────────────────────────────────────────────────────────────┘

문제점:
❌ 서비스마다 공인 IP 필요 → 비용 증가 (IP당 월 $3~5)
❌ IP 주소 외우기 어려움
❌ 서비스 10개면 IP 10개 필요
```

#### 방법 3: Ingress (권장)

```
┌──────────────────────────────────────────────────────────────┐
│  IP 1개로 모든 서비스 접근                                    │
│                                                              │
│  http://myapp.com/         ──►  web-ui                       │
│  http://myapp.com/api      ──►  web-api                      │
│  http://myapp.com/mcp      ──►  web-mcp                      │
│                                                              │
│  또는 도메인으로:                                             │
│  http://app.mysite.com     ──►  web-ui                       │
│  http://api.mysite.com     ──►  web-api                      │
└──────────────────────────────────────────────────────────────┘

장점:
✅ IP 1개로 여러 서비스 (비용 절감)
✅ 경로(Path) 기반 라우팅
✅ 도메인(Host) 기반 라우팅
✅ HTTPS 인증서 중앙 관리
✅ 로드밸런싱
```

### 비용 비교

| 방식 | 서비스 5개일 때 | 월 비용 (예상) |
|------|---------------|---------------|
| LoadBalancer | IP 5개 필요 | $15~25 |
| Ingress | IP 1개 | $3~5 |

---

## 3. 핵심 개념

### Ingress vs Ingress Controller

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Ingress (리소스)              Ingress Controller (실행자)   │
│  ┌───────────────────┐        ┌───────────────────┐         │
│  │ "설정 문서"        │        │ "실제 라우터"      │         │
│  │                   │        │                   │         │
│  │ - /api → web-api  │ ────►  │ NGINX / Traefik   │         │
│  │ - /    → web-ui   │        │ 등 실제 동작      │         │
│  └───────────────────┘        └───────────────────┘         │
│                                                             │
│  YAML로 정의                   Pod로 실행됨                  │
│  (선언적)                      (실제 트래픽 처리)            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**비유:**
- Ingress = 교통 규칙 (설정)
- Ingress Controller = 신호등/교통경찰 (실행)

### 라우팅 방식 2가지

#### 1. Path 기반 라우팅

```yaml
rules:
  - http:
      paths:
        - path: /api
          backend:
            service:
              name: web-api
        - path: /
          backend:
            service:
              name: web-ui
```

```
http://myapp.com/api/users  ──►  web-api 서비스
http://myapp.com/dashboard  ──►  web-ui 서비스
```

#### 2. Host 기반 라우팅

```yaml
rules:
  - host: api.myapp.com
    http:
      paths:
        - path: /
          backend:
            service:
              name: web-api
  - host: www.myapp.com
    http:
      paths:
        - path: /
          backend:
            service:
              name: web-ui
```

```
http://api.myapp.com/users  ──►  web-api 서비스
http://www.myapp.com/       ──►  web-ui 서비스
```

---

## 4. 현재 프로젝트 Ingress 설정

### 파일 구조

```
kubernetes/
├── base/
│   └── ingress/
│       ├── kustomization.yaml
│       └── ingress.yaml           # 기본 Ingress 설정
└── overlays/
    └── prd/
        └── ingress-patch.yaml     # PRD용 패치 (TLS 등)
```

### 현재 라우팅 규칙

```yaml
# kubernetes/base/ingress/ingress.yaml
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          # 1. NextAuth API (프론트에서 처리)
          - path: /api/auth
            pathType: Prefix
            backend:
              service:
                name: cms-web-ui
                port:
                  number: 15001

          # 2. Backend API
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: cms-web-api
                port:
                  number: 18001

          # 3. MCP 서버
          - path: /mcp
            pathType: Prefix
            backend:
              service:
                name: cms-web-mcp
                port:
                  number: 18002

          # 4. 파일 업로드/다운로드
          - path: /files
            pathType: Prefix
            backend:
              service:
                name: cms-web-api
                port:
                  number: 18001

          # 5. 프론트엔드 (기본)
          - path: /
            pathType: Prefix
            backend:
              service:
                name: cms-web-ui
                port:
                  number: 15001
```

### 시각적 라우팅 맵

```
┌─────────────────────────────────────────────────────────────────┐
│                        Ingress Controller                       │
│                     (http://20.249.205.59)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  요청 URL                          라우팅 대상                  │
│  ─────────────────────────────────────────────────────────      │
│                                                                 │
│  /api/auth/*  ─────────────────►  cms-web-ui:15001              │
│  (로그인/로그아웃)                  (NextAuth 처리)              │
│                                                                 │
│  /api/*       ─────────────────►  cms-web-api:18001             │
│  (REST API)                        (FastAPI Backend)            │
│                                                                 │
│  /mcp/*       ─────────────────►  cms-web-mcp:18002             │
│  (MCP 프로토콜)                    (MCP Server)                  │
│                                                                 │
│  /files/*     ─────────────────►  cms-web-api:18001             │
│  (파일 업/다운로드)                 (파일 처리)                   │
│                                                                 │
│  /*           ─────────────────►  cms-web-ui:15001              │
│  (그 외 모든 요청)                  (Next.js Frontend)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 경로 매칭 순서 (중요!)

```
요청: /api/auth/signin

1. /api/auth/* 에 매칭? ✅ → cms-web-ui로 라우팅
2. /api/* 는 체크 안 함 (이미 매칭됨)

요청: /api/users

1. /api/auth/* 에 매칭? ❌
2. /api/* 에 매칭? ✅ → cms-web-api로 라우팅

⚠️ 더 구체적인 경로를 먼저 정의해야 함!
   /api/auth 가 /api 보다 위에 있어야 함
```

---

## 5. Annotations 상세 설명

### 현재 설정된 Annotations

```yaml
annotations:
  # Ingress Controller 종류
  kubernetes.io/ingress.class: nginx

  # 업로드 파일 크기 제한 (100MB)
  nginx.ingress.kubernetes.io/proxy-body-size: "100m"

  # 타임아웃 설정 (3600초 = 1시간)
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"

  # CORS 설정
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "*"
  nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS, PATCH"
  nginx.ingress.kubernetes.io/cors-allow-headers: "..."
```

### 각 설정의 의미

| Annotation | 값 | 왜 필요해? |
|------------|-----|----------|
| `proxy-body-size` | 100m | 큰 파일 업로드 허용 (소스코드 zip 등) |
| `proxy-read-timeout` | 3600 | 변환 작업이 오래 걸릴 수 있음 (1시간) |
| `enable-cors` | true | 프론트엔드 ↔ 백엔드 도메인 다를 때 허용 |
| `cors-allow-origin` | * | 모든 도메인에서 API 호출 허용 |

### 자주 쓰는 다른 Annotations

```yaml
# HTTPS 강제 리다이렉트
nginx.ingress.kubernetes.io/ssl-redirect: "true"

# 업스트림 연결 유지
nginx.ingress.kubernetes.io/upstream-keepalive-connections: "100"

# 요청 속도 제한 (DDoS 방어)
nginx.ingress.kubernetes.io/limit-rps: "100"

# 경로 재작성
nginx.ingress.kubernetes.io/rewrite-target: /
```

---

## 6. HTTPS 설정 (TLS)

### 현재 상태 (HTTP)

```yaml
# PRD 환경에서 TLS 비활성화 상태
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "false"
```

### HTTPS 활성화 방법

#### 방법 1: 수동 인증서

```yaml
# 1. Secret 생성
kubectl create secret tls my-tls-secret \
  --cert=fullchain.pem \
  --key=privkey.pem \
  -n prd

# 2. Ingress에 TLS 설정
spec:
  tls:
    - hosts:
        - myapp.com
      secretName: my-tls-secret
  rules:
    - host: myapp.com
      http:
        paths:
          ...
```

#### 방법 2: cert-manager (자동 갱신)

```yaml
# 1. cert-manager 설치
# 2. Ingress에 annotation 추가
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"

spec:
  tls:
    - hosts:
        - myapp.com
      secretName: myapp-tls-secret  # 자동 생성됨
```

### PRD 환경 TLS 패치 예시

```yaml
# overlays/prd/ingress-patch.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auto-builder-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - cloudtr.kubepia.net
      secretName: cloudtr-tls-secret
  rules:
    - host: cloudtr.kubepia.net
      http:
        paths:
          ...
```

---

## 7. Service와의 관계

### 전체 흐름

```
┌─────────────────────────────────────────────────────────────────┐
│                         트래픽 흐름                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  외부 요청                                                       │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────┐                                            │
│  │  LoadBalancer   │  ← Ingress Controller의 Service            │
│  │  (공인 IP)      │    type: LoadBalancer                      │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │    Ingress      │  ← 라우팅 규칙 (YAML)                       │
│  │   Controller    │    어떤 경로 → 어떤 Service                 │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │    Service      │  ← ClusterIP (내부 접근용)                  │
│  │  (ClusterIP)    │    Pod들을 묶어주는 역할                    │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │      Pod        │  ← 실제 앱 컨테이너                         │
│  └─────────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 현재 프로젝트 Service 구조

```yaml
# cms-web-api Service
apiVersion: v1
kind: Service
metadata:
  name: cms-web-api
spec:
  type: ClusterIP          # 내부 접근만 (Ingress 통해 외부 노출)
  ports:
    - port: 18001          # Service 포트
      targetPort: 18001    # Pod 포트
  selector:
    app: cms-web-api       # 이 라벨 가진 Pod로 라우팅
```

**연결 관계:**
```
Ingress                  Service                  Pod
─────────────────────────────────────────────────────────────
path: /api          ──►  cms-web-api:18001  ──►  cms-web-api Pod
                         (ClusterIP)              (app: cms-web-api)
```

---

## 8. 실제 적용 방법

### Step 1: Ingress Controller 설치 확인

```bash
# AKS에서 NGINX Ingress Controller 확인
kubectl get pods -n ingress-nginx

# 또는 ingress-basic 네임스페이스
kubectl get pods -n ingress-basic

# Ingress Controller Service 확인 (외부 IP)
kubectl get svc -n ingress-nginx
```

**출력 예시:**
```
NAME                       TYPE           EXTERNAL-IP     PORT(S)
ingress-nginx-controller   LoadBalancer   20.249.205.59   80:30080,443:30443
```

### Step 2: Ingress 배포

```bash
# Kustomize로 배포 (base에 포함됨)
kubectl apply -k kubernetes/overlays/dev

# 또는 Ingress만 개별 적용
kubectl apply -f kubernetes/base/ingress/ingress.yaml -n dev
```

### Step 3: 적용 확인

```bash
# Ingress 목록 확인
kubectl get ingress -n dev

# 결과 예시
NAME                    CLASS   HOSTS   ADDRESS         PORTS   AGE
auto-builder-ingress    nginx   *       20.249.205.59   80      5m
```

### Step 4: 상세 정보 확인

```bash
kubectl describe ingress auto-builder-ingress -n dev
```

**출력 예시:**
```
Name:             auto-builder-ingress
Namespace:        dev
Address:          20.249.205.59
Ingress Class:    nginx
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /api/auth   cms-web-ui:15001 (10.244.0.5:15001)
              /api        cms-web-api:18001 (10.244.0.6:18001)
              /mcp        cms-web-mcp:18002 (10.244.0.7:18002)
              /files      cms-web-api:18001 (10.244.0.6:18001)
              /           cms-web-ui:15001 (10.244.0.5:15001)
```

### Step 5: 라우팅 테스트

```bash
# 프론트엔드 접근
curl http://20.249.205.59/

# API 접근
curl http://20.249.205.59/api/health

# MCP 접근
curl http://20.249.205.59/mcp/health
```

---

## 9. 트러블슈팅

### 문제 1: 404 Not Found

```bash
# 증상
curl http://20.249.205.59/api/users
# 404 Not Found

# 원인 확인
kubectl describe ingress auto-builder-ingress -n dev

# 체크 포인트:
# 1. Service 이름이 맞는지
# 2. Service 포트가 맞는지
# 3. Service가 존재하는지
kubectl get svc -n dev
```

### 문제 2: 502 Bad Gateway

```bash
# 증상: Ingress는 있는데 502 에러

# 원인: Backend Pod가 없거나 준비 안 됨
kubectl get pods -n dev

# Pod 상태 확인
kubectl describe pod cms-web-api-xxx -n dev

# Service Endpoints 확인
kubectl get endpoints cms-web-api -n dev
# Endpoints가 비어있으면 Pod와 연결 안 됨
```

### 문제 3: 413 Request Entity Too Large

```bash
# 증상: 파일 업로드 시 에러

# 해결: proxy-body-size 증가
annotations:
  nginx.ingress.kubernetes.io/proxy-body-size: "500m"
```

### 문제 4: 504 Gateway Timeout

```bash
# 증상: 오래 걸리는 요청에서 타임아웃

# 해결: timeout 증가
annotations:
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

### 문제 5: CORS 에러

```bash
# 증상: 브라우저에서 CORS 에러

# 해결: CORS annotation 추가
annotations:
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "https://myapp.com"
```

---

## 10. 요약

### Ingress 없을 때 vs 있을 때

| 구분 | Ingress 없음 | Ingress 있음 |
|------|-------------|-------------|
| 외부 접근 | NodePort 또는 LoadBalancer | 경로 기반 라우팅 |
| IP 개수 | 서비스 수만큼 | 1개 |
| 비용 | 높음 | 낮음 |
| HTTPS | 각 서비스에서 처리 | 중앙 관리 |
| 라우팅 | 불가능 | 경로/도메인 기반 |

### 현재 프로젝트 라우팅 정리

| 경로 | 서비스 | 용도 |
|------|--------|------|
| `/api/auth/*` | cms-web-ui:15001 | NextAuth 로그인 |
| `/api/*` | cms-web-api:18001 | REST API |
| `/mcp/*` | cms-web-mcp:18002 | MCP 프로토콜 |
| `/files/*` | cms-web-api:18001 | 파일 처리 |
| `/*` | cms-web-ui:15001 | 프론트엔드 |

### 핵심 정리

```
1. Ingress = URL 경로 기반 라우팅
2. IP 1개로 여러 서비스 노출 (비용 절감)
3. HTTPS 인증서 중앙 관리
4. Ingress Controller가 실제 동작 (NGINX)
5. 더 구체적인 경로를 먼저 정의해야 함
```

### 적용 명령어 요약

```bash
# 1. Ingress Controller 확인
kubectl get pods -n ingress-nginx

# 2. 배포
kubectl apply -k kubernetes/overlays/dev

# 3. 확인
kubectl get ingress -n dev

# 4. 상세 확인
kubectl describe ingress auto-builder-ingress -n dev

# 5. 테스트
curl http://<EXTERNAL-IP>/api/health
```

---

## 11. 관련 문서

- [Kubernetes Ingress 공식 문서](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [AKS Ingress 가이드](https://learn.microsoft.com/ko-kr/azure/aks/ingress-basic)
- [[학습] HPA](./[학습]%20HPA.md)
- [[학습] PDB](./[학습]%20PDB.md)
