# ADR-M004: 데이터베이스 선택

> **Status**: 승인됨
> **Date**: 2025-12
> **Category**: Migration - Database

---

## 상태
✅ **승인됨** (2025-12)

## 컨텍스트
기존에는 Docker Compose로 PostgreSQL 컨테이너를 직접 운영. K8s 마이그레이션 시 DB 운영 방식 결정 필요

## 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. K8s 내 StatefulSet** | 완전한 제어, 비용 절감 | 운영 부담, 백업/복구 직접 구현 |
| **B. Azure Database for PostgreSQL (Single Server)** | 관리형 | 곧 지원 종료 예정 |
| **C. Azure Database for PostgreSQL (Flexible Server)** | 관리형, 최신, 유연한 설정 | 비용 |
| **D. Azure Cosmos DB (PostgreSQL)** | 글로벌 분산 | 과도한 스펙, 높은 비용 |

## 결정
**옵션 C: Azure Database for PostgreSQL Flexible Server**

## 근거
1. **관리형 서비스**: 백업, 패치, HA 자동 관리
2. **Flexible Server**: Single Server 대비 최신 기능, 더 나은 가격/성능
3. **Azure 통합**: VNet 통합, AAD 인증 지원
4. **확장성**: 필요시 스펙 변경 용이

## 결과
```
DEV: psql-cloudtr-dev.postgres.database.azure.com
PRD: psql-cloudtr-prd.postgres.database.azure.com
SKU: Burstable B1ms (1 vCore, 2GB)
Storage: 32GB
```

## 보안 설정
- AKS 서브넷에서만 접근 허용 (방화벽 규칙)
- SSL 필수 (`sslmode=require`)
- 비밀번호는 Key Vault에 저장

## 관련 파일
```
azure/scripts/04-postgresql.sh
```

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
