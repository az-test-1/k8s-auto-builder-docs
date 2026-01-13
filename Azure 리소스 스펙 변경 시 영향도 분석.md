# Azure 리소스 스펙 변경 시 영향도 분석

> 00-env.sh 설정값 변경 시 각 리소스별 영향도 및 다운타임 분석

---

## 1. AKS 클러스터

| 설정 | 현재값 | 변경 영향 | 다운타임 |
|------|--------|----------|---------|
| `AKS_NODE_VM_SIZE` | Standard_B2ms | ❌ **변경 불가** - 클러스터 재생성 필요 | 전체 중단 |
| `AKS_NODE_COUNT` | 1 | ✅ 런타임 변경 가능 | 없음 |
| `AKS_MIN_COUNT` | 1 | ✅ 런타임 변경 가능 | 없음 |
| `AKS_MAX_COUNT` | 3 | ✅ 런타임 변경 가능 | 없음 |
| `K8S_VERSION` | 1.30 | ⚠️ 업그레이드만 가능 (다운그레이드 불가) | 롤링 업데이트 |

### VM Size 변경 방법 (노드풀 추가)

```bash
# 기존 노드풀 유지하면서 새 노드풀 추가
az aks nodepool add -g rg-cloudtr-prd --cluster-name aks-cloudtr-prd \
  -n newpool --node-vm-size Standard_D2s_v3 --node-count 2

# 기존 노드풀의 워크로드를 새 노드풀로 이동
kubectl cordon <old-node>
kubectl drain <old-node> --ignore-daemonsets --delete-emptydir-data

# 기존 노드풀 삭제
az aks nodepool delete -g rg-cloudtr-prd --cluster-name aks-cloudtr-prd -n nodepool1
```

### Autoscaling 변경

```bash
# 런타임에 변경 가능
az aks update -g rg-cloudtr-prd -n aks-cloudtr-prd \
  --update-cluster-autoscaler \
  --min-count 2 \
  --max-count 5
```

### Kubernetes 버전 업그레이드

```bash
# 사용 가능한 버전 확인
az aks get-upgrades -g rg-cloudtr-prd -n aks-cloudtr-prd

# 업그레이드 (롤링 업데이트 - 다운타임 최소화)
az aks upgrade -g rg-cloudtr-prd -n aks-cloudtr-prd --kubernetes-version 1.31
```

---

## 2. PostgreSQL Flexible Server

| 설정 | 현재값 | 변경 영향 | 다운타임 |
|------|--------|----------|---------|
| `POSTGRES_SKU` | Standard_B1ms | ⚠️ 스케일업/다운 가능 | **1-5분 다운타임** |
| `POSTGRES_STORAGE_SIZE` | 32GB | ⚠️ **증가만 가능** (축소 불가) | 없음 |
| `POSTGRES_ADMIN_USER` | ab | ❌ **변경 불가** - 재생성 필요 | 전체 중단 |
| `POSTGRES_DB` | auto_builder | ✅ 추가 DB 생성 가능 | 없음 |

### SKU 변경 (스케일업/다운)

```bash
# 스케일업 (1-5분 다운타임 발생)
az postgres flexible-server update -g rg-cloudtr-prd -n psql-cloudtr-prd \
  --sku-name Standard_B2ms

# 스케일다운
az postgres flexible-server update -g rg-cloudtr-prd -n psql-cloudtr-prd \
  --sku-name Standard_B1ms
```

### 스토리지 증가

```bash
# 스토리지 증가 (다운타임 없음, 축소 불가!)
az postgres flexible-server update -g rg-cloudtr-prd -n psql-cloudtr-prd \
  --storage-size 64
```

### 사용 가능한 SKU 목록

| SKU | vCPU | Memory | 용도 |
|-----|------|--------|------|
| Standard_B1ms | 1 | 2GB | 개발/테스트 |
| Standard_B2ms | 2 | 4GB | 소규모 프로덕션 |
| Standard_B2s | 2 | 4GB | 소규모 프로덕션 |
| Standard_D2s_v3 | 2 | 8GB | 프로덕션 |
| Standard_D4s_v3 | 4 | 16GB | 대규모 프로덕션 |

---

## 3. Storage Account (Azure Files)

| 설정 | 현재값 | 변경 영향 | 다운타임 |
|------|--------|----------|---------|
| `STORAGE_ACCOUNT` | stcloudtrprd | ❌ **변경 불가** - 재생성 필요 | 데이터 이전 필요 |
| `STORAGE_SHARE` | cms-data | ❌ 이름 변경 불가 | - |
| `STORAGE_QUOTA` | 100GB | ✅ 증가/감소 가능 | 없음 |

### Quota 변경

```bash
# Quota 증가
az storage share-rm update -g rg-cloudtr-prd \
  --storage-account stcloudtrprd --name cms-data --quota 200

# Quota 감소 (사용량보다 작게 설정 불가)
az storage share-rm update -g rg-cloudtr-prd \
  --storage-account stcloudtrprd --name cms-data --quota 100
```

### 주의사항

- Premium FileStorage는 최소 100GB
- Standard FileStorage는 최소 1GB
- 이름 변경 시 새 Storage Account 생성 후 데이터 마이그레이션 필요

---

## 4. Key Vault

| 설정 | 현재값 | 변경 영향 | 다운타임 |
|------|--------|----------|---------|
| `KEYVAULT_NAME` | kv-cloudtr-prd | ❌ **변경 불가** - 재생성 필요 | 전체 재설정 필요 |

### Secrets 관리 (런타임 변경 가능)

```bash
# Secret 추가/수정 (다운타임 없음)
az keyvault secret set --vault-name kv-cloudtr-prd --name NEW-SECRET --value "value"

# Secret 삭제
az keyvault secret delete --vault-name kv-cloudtr-prd --name OLD-SECRET

# Secret 목록 확인
az keyvault secret list --vault-name kv-cloudtr-prd --query "[].name" -o tsv
```

### 주의사항

- Key Vault 이름은 Azure 전역에서 고유해야 함
- 삭제된 Key Vault는 90일간 복구 가능 (Soft Delete)
- 같은 이름으로 재생성하려면 Purge 필요

---

## 5. ACR (Container Registry)

| 설정 | 현재값 | 변경 영향 | 다운타임 |
|------|--------|----------|---------|
| `ACR_NAME` | acraz01cloudtr | ❌ **변경 불가** - 재생성 필요 | 모든 이미지 재푸시 필요 |

### SKU 변경 (가능)

```bash
# Basic → Standard → Premium 업그레이드 가능
az acr update -n acraz01cloudtr --sku Standard
```

| SKU | 스토리지 | 기능 | 월 비용 |
|-----|---------|------|--------|
| Basic | 10GB | 기본 | ~$5 |
| Standard | 100GB | Webhook, 복제 | ~$20 |
| Premium | 500GB | Geo-복제, Private Link | ~$50+ |

---

## 6. 네트워크 (VNet/Subnet)

| 설정 | 현재값 | 변경 영향 | 다운타임 |
|------|--------|----------|---------|
| `VNET_NAME` | vnet-cloudtr | ❌ **변경 불가** | 전체 재생성 |
| `VNET_ADDRESS_PREFIX` | 10.0.0.0/8 | ❌ **변경 불가** | 전체 재생성 |
| `AKS_SUBNET_PREFIX` | 10.240.0.0/16 | ❌ **변경 불가** | AKS 재생성 필요 |
| `POSTGRES_SUBNET_PREFIX` | 10.241.0.0/24 | ❌ **변경 불가** | PostgreSQL 재생성 필요 |

### 주의사항

- 네트워크 CIDR는 생성 후 변경 불가
- 서브넷에 리소스가 있으면 삭제 불가
- 처음 설계 시 충분히 큰 범위로 할당 필요

---

## 요약: 변경 가능 여부 매트릭스

| 리소스 | 쉽게 변경 가능 | 다운타임 필요 | 재생성 필요 |
|--------|---------------|--------------|------------|
| **AKS** | 노드 수, Autoscaling | K8s 버전 업그레이드 | VM Size, 네트워크 |
| **PostgreSQL** | 스토리지 증가 | SKU 변경 | Admin User, 스토리지 축소 |
| **Storage** | Quota | - | Account Name, Share Name |
| **Key Vault** | Secrets 추가/수정 | - | Vault Name |
| **ACR** | SKU 업그레이드 | - | Registry Name |
| **Network** | - | - | 모든 CIDR, VNet Name |

---

## 권장사항

### 초기 설계 시

1. **VM Size는 넉넉하게**: 나중에 변경하려면 노드풀 교체 필요
2. **네트워크 CIDR 여유있게**: 변경 불가능하므로 확장 고려
3. **PostgreSQL 스토리지**: 축소 불가하므로 적절히 시작
4. **이름 규칙 확정**: 대부분의 리소스 이름은 변경 불가

### 스케일링 전략

1. **AKS**: Autoscaling 활용, 필요시 노드풀 추가
2. **PostgreSQL**: Read Replica 추가로 읽기 부하 분산
3. **Storage**: Premium tier로 IOPS 확보

### 비용 최적화

1. **Burstable SKU (B-series)**: 개발/테스트 환경에 적합
2. **Reserved Instances**: 1-3년 약정으로 최대 72% 할인
3. **Autoscaling**: 사용량에 따른 자동 조절

---

## 변경 이력

| 날짜 | 변경 내용 |
|------|----------|
| 2026-01-12 | 최초 작성 |
