# 대규모 처리를 위한 스토리지 아키텍처 설계

> 작성일: 2026-01-08
> 목표: 다수 사용자의 대량 빌드 처리 지원
> 상태: **구현 진행 중** (Phase 2 완료)

---

## 1. 목표 정의

### 1.1 목표 수치

| 항목 | 현재 | 목표 | 비고 |
|------|-----:|-----:|------|
| 동시 사용자 | ~10명 | **100명+** | 10배 |
| 동시 빌드 | 1-2개 | **20-50개** | 25배 |
| 일일 빌드 | ~50건 | **500-1,000건** | 20배 |
| 응답 시간 | - | **< 3초** | API |
| 빌드 시간 | - | **현재 대비 유지** | 스케일 시에도 |

### 1.2 현재 구성의 병목 예측

```
동시 빌드 20개 시나리오:

Azure Files (현재):
├── 총 IOPS: 3,000 (100GB Premium 기준)
├── Pod당 IOPS: 3,000 / 20 = 150 IOPS
├── 예상 빌드 시간: 기존 대비 3-5배 증가
└── 결과: ❌ 심각한 성능 저하

필요한 개선:
├── 독립적인 I/O (Pod 간 경합 제거)
├── 무제한 확장 가능한 스토리지
└── 빌드 작업의 로컬 처리
```

---

## 2. 권장 아키텍처: 하이브리드 스토리지

### 2.1 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────────┐
│                         하이브리드 스토리지 아키텍처                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      Azure Blob Storage (Primary)                    │
│                    ┌─────────────────────────────┐                  │
│                    │  uploads/      (소스 ZIP)   │                  │
│                    │  projects/     (빌드 결과)  │                  │
│                    │  artifacts/    (산출물)     │                  │
│                    └─────────────────────────────┘                  │
│                              │                                       │
│                         REST API                                     │
└─────────────────────────────────────────────────────────────────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
           ▼                   ▼                   ▼
     ┌──────────┐        ┌──────────┐        ┌──────────┐
     │ web-api  │        │ worker-1 │        │ worker-N │
     │ Pod      │        │ Pod      │        │ Pod      │
     │          │        │          │        │          │
     │ Blob SDK │        │ Blob SDK │        │ Blob SDK │
     └──────────┘        └──────────┘        └──────────┘
                               │
                         K8s Job 생성
                               │
     ┌─────────────────────────┼─────────────────────────┐
     │                         │                         │
     ▼                         ▼                         ▼
┌──────────┐             ┌──────────┐             ┌──────────┐
│ Job-1    │             │ Job-2    │             │ Job-N    │
│ emptyDir │             │ emptyDir │             │ emptyDir │
│ (로컬SSD)│             │ (로컬SSD)│             │ (로컬SSD)│
│          │             │          │             │          │
│ 1.다운로드│             │ 1.다운로드│             │ 1.다운로드│
│ 2.빌드   │             │ 2.빌드   │             │ 2.빌드   │
│ 3.업로드 │             │ 3.업로드 │             │ 3.업로드 │
└──────────┘             └──────────┘             └──────────┘
     │                         │                         │
     └─────────────────────────┴─────────────────────────┘
                               │
                    각 Job이 독립적으로 작업
                    (I/O 경합 없음!)

┌─────────────────────────────────────────────────────────────────────┐
│                    Azure Files PVC (Cache Only)                      │
│                    ┌─────────────────────────────┐                  │
│                    │  .m2/repository (Maven)     │   10Gi           │
│                    │  .gradle/caches (Gradle)    │   캐시 전용      │
│                    └─────────────────────────────┘                  │
│                    손실되어도 무방, 재다운로드 가능                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 데이터 흐름

```
사용자 요청 → 빌드 완료까지의 흐름:

1. 소스 업로드
   User → API → Blob Storage (uploads/{project_id}/source.zip)
                     │
                     ▼ 비동기 (빠름)
                  완료 응답

2. 빌드 시작
   API → Celery Worker → K8s Job 생성
                              │
                              ▼
                    ┌─────────────────┐
                    │   K8s Job Pod   │
                    │   (emptyDir)    │
                    └─────────────────┘
                              │
3. 빌드 실행              ▼
   ┌──────────────────────────────────────────┐
   │ 1. Blob에서 소스 다운로드 → /tmp/source  │  (2-5초)
   │ 2. 캐시 PVC 마운트 확인 → /cache/.m2    │
   │ 3. 로컬 SSD에서 빌드 수행               │  (빠름!)
   │ 4. 결과물 Blob에 업로드                 │  (2-5초)
   │ 5. 로그 Blob에 업로드                   │
   └──────────────────────────────────────────┘
                              │
4. 결과 조회               ▼
   User → API → Blob Storage (projects/{project_id}/result.zip)
                     │
                     ▼ Pre-signed URL
                  직접 다운로드 (CDN 가능)
```

---

## 3. 상세 구성 요소

### 3.1 Azure Blob Storage 컨테이너 구조

```
Storage Account: stcloudtrcms
│
├── Container: uploads
│   └── {project_id}/
│       └── source.zip              # 업로드된 소스코드
│
├── Container: projects
│   └── {project_id}/
│       ├── source/                 # 압축 해제된 소스 (선택)
│       ├── build/
│       │   └── result.zip          # 빌드 결과물
│       └── logs/
│           ├── build.log           # 빌드 로그
│           └── migration.log       # 마이그레이션 로그
│
├── Container: artifacts
│   └── {project_id}/
│       └── {version}/
│           └── app.jar             # 최종 산출물
│
└── Container: cache (선택적)
    └── maven/
        └── repository/             # Maven 캐시 백업
```

### 3.2 PVC 구성 (캐시 전용)

```yaml
# 캐시 전용 PVC (크기 축소)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: build-cache-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-csi-premium
  resources:
    requests:
      storage: 20Gi  # 100Gi → 20Gi (80% 축소)
```

**마운트 구조:**
```
/cache/                    # 캐시 PVC (20Gi)
├── .m2/repository/        # Maven 캐시
└── .gradle/caches/        # Gradle 캐시

/tmp/                      # emptyDir (로컬 SSD)
├── source/                # 다운로드된 소스
├── build/                 # 빌드 작업 공간
└── output/                # 빌드 결과물
```

### 3.3 K8s Job 수정

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: build-job-{project_id}
spec:
  template:
    spec:
      containers:
        - name: builder
          image: acr/builder:latest
          env:
            # Blob Storage 연결 정보
            - name: AZURE_STORAGE_CONNECTION_STRING
              valueFrom:
                secretKeyRef:
                  name: blob-secrets
                  key: connection-string
            - name: PROJECT_ID
              value: "{project_id}"
          volumeMounts:
            # 캐시만 PVC 사용
            - name: cache
              mountPath: /cache
            # 빌드 작업은 로컬 emptyDir
            - name: workspace
              mountPath: /tmp/build
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
              ephemeral-storage: "10Gi"  # 로컬 SSD
            limits:
              cpu: "2"
              memory: "4Gi"
              ephemeral-storage: "20Gi"
      volumes:
        - name: cache
          persistentVolumeClaim:
            claimName: build-cache-pvc
        - name: workspace
          emptyDir:
            sizeLimit: 20Gi  # Pod 로컬 스토리지
```

---

## 4. 성능 비교 (예측)

### 4.1 단일 빌드 성능

| 단계 | 현재 (PVC) | 개선 후 (Blob+Local) | 차이 |
|------|----------:|--------------------:|-----:|
| 소스 준비 | 5초 | 3초 | -40% |
| 빌드 실행 | 60초 | 45초 | -25% |
| 결과 저장 | 5초 | 3초 | -40% |
| **총 시간** | **70초** | **51초** | **-27%** |

### 4.2 동시 빌드 성능 (핵심!)

| 동시 빌드 수 | 현재 (PVC) | 개선 후 | 개선율 |
|------------:|----------:|---------:|-------:|
| 1개 | 70초 | 51초 | -27% |
| 5개 | 100초 | 55초 | -45% |
| 10개 | 150초 | 55초 | **-63%** |
| 20개 | 250초 | 60초 | **-76%** |
| 50개 | 500초+ | 70초 | **-86%** |

```
그래프: 동시 빌드 수에 따른 빌드 시간

시간(초)
500 │                                    ╭─── 현재 (PVC)
    │                               ╭────╯
400 │                          ╭────╯
    │                     ╭────╯
300 │                ╭────╯
    │           ╭────╯
200 │      ╭────╯
    │ ╭────╯
100 │─╯────────────────────────────────── 개선 후 (Blob+Local)
    │
  0 └─────┬─────┬─────┬─────┬─────┬────
          1     5    10    20    50   동시 빌드 수

핵심: 개선 후에는 동시 빌드 수가 늘어도 시간이 거의 일정!
```

### 4.3 비용 비교

| 항목 | 현재 | 개선 후 | 차이 |
|------|-----:|-------:|-----:|
| PVC (Azure Files) | $17.50 | $3.50 (20Gi) | -80% |
| Blob Storage | $0 | $5.00 (예상) | +$5 |
| 트랜잭션 | 포함 | $2.00 (예상) | +$2 |
| **월 총 비용** | **$17.50** | **$10.50** | **-40%** |

### 4.4 확장성 비교

| 항목 | 현재 | 개선 후 |
|------|------|---------|
| 최대 동시 빌드 | ~10개 (성능 저하) | **무제한** |
| 스토리지 확장 | 수동 (PVC 수정) | **자동** |
| Pod 스케일 아웃 | 제한적 | **자유로움** |
| 지역 확장 | 단일 리전 | **다중 리전 가능** |

---

## 5. 구현 계획

### 5.1 Phase 1: Blob Storage 인프라 (1단계)

```bash
# 1. Storage Account 생성
az storage account create \
  --name stcloudtrcms \
  --resource-group rg-cloudtr \
  --location koreacentral \
  --sku Standard_LRS \
  --kind StorageV2

# 2. Container 생성
az storage container create --name uploads --account-name stcloudtrcms
az storage container create --name projects --account-name stcloudtrcms
az storage container create --name artifacts --account-name stcloudtrcms

# 3. K8s Secret 생성
kubectl create secret generic blob-secrets \
  --from-literal=connection-string="DefaultEndpoints..." \
  -n auto-builder
```

### 5.2 Phase 2: 애플리케이션 코드 수정 (2단계)

```python
# 새로운 Storage Service 추가
# src/app/services/blob_storage_service.py

from azure.storage.blob import BlobServiceClient
import os

class BlobStorageService:
    def __init__(self):
        conn_str = os.getenv("AZURE_STORAGE_CONNECTION_STRING")
        self.client = BlobServiceClient.from_connection_string(conn_str)

    async def upload_source(self, project_id: str, file: UploadFile):
        """소스 파일 업로드"""
        blob_client = self.client.get_blob_client(
            container="uploads",
            blob=f"{project_id}/source.zip"
        )
        await blob_client.upload_blob(file.file, overwrite=True)
        return blob_client.url

    async def download_source(self, project_id: str, local_path: str):
        """빌드 Job에서 소스 다운로드"""
        blob_client = self.client.get_blob_client(
            container="uploads",
            blob=f"{project_id}/source.zip"
        )
        with open(local_path, "wb") as f:
            data = await blob_client.download_blob()
            f.write(data.readall())

    async def upload_result(self, project_id: str, local_path: str):
        """빌드 결과 업로드"""
        blob_client = self.client.get_blob_client(
            container="projects",
            blob=f"{project_id}/build/result.zip"
        )
        with open(local_path, "rb") as f:
            await blob_client.upload_blob(f, overwrite=True)

    def get_download_url(self, project_id: str) -> str:
        """SAS URL 생성 (직접 다운로드용)"""
        blob_client = self.client.get_blob_client(
            container="projects",
            blob=f"{project_id}/build/result.zip"
        )
        sas_token = generate_blob_sas(...)  # 1시간 유효
        return f"{blob_client.url}?{sas_token}"
```

### 5.3 Phase 3: K8s Job 수정 (3단계)

```yaml
# kubernetes/base/cms/job-template.yaml 수정
spec:
  template:
    spec:
      initContainers:
        # 소스 다운로드 (Blob → Local)
        - name: download-source
          image: mcr.microsoft.com/azure-cli
          command:
            - /bin/sh
            - -c
            - |
              az storage blob download \
                --container-name uploads \
                --name ${PROJECT_ID}/source.zip \
                --file /workspace/source.zip \
                --connection-string "${AZURE_STORAGE_CONNECTION_STRING}"
              unzip /workspace/source.zip -d /workspace/source
          volumeMounts:
            - name: workspace
              mountPath: /workspace

      containers:
        - name: builder
          # 로컬에서 빌드 수행
          volumeMounts:
            - name: workspace
              mountPath: /workspace
            - name: cache
              mountPath: /cache

      # 빌드 완료 후 결과 업로드 (sidecar 또는 postStop)
```

### 5.4 Phase 4: PVC 축소 (4단계)

```yaml
# 기존: 100Gi → 변경: 20Gi (캐시 전용)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: build-cache-pvc  # 이름 변경
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-csi-premium
  resources:
    requests:
      storage: 20Gi
```

---

## 6. 마이그레이션 체크리스트

### 6.1 준비 단계

- [ ] Azure Storage Account 생성
- [ ] Blob Container 생성 (uploads, projects, artifacts)
- [x] Connection String을 K8s Secret으로 저장 (템플릿 추가됨)
- [x] azure-storage-blob Python 패키지 추가 ✅

### 6.2 개발 단계

- [x] BlobStorageService 클래스 구현 ✅
- [ ] 파일 업로드 API 수정 (PVC → Blob)
- [ ] 파일 다운로드 API 수정 (SAS URL 반환)
- [ ] K8s Job 템플릿 수정 (initContainer 추가)
- [ ] 빌드 스크립트 수정 (로컬 작업 → Blob 업로드)

### 6.3 구현 완료 내역 (2026-01-08)

**파일 변경 목록:**

```
수정된 파일:
├── requirements.txt
│   └── azure-storage-blob==12.24.0, azure-identity==1.19.0 추가
│
├── src/app/config/config.py
│   └── AZURE_STORAGE_* 환경변수 설정 추가
│
├── src/app/domains/cms/file/model/file_model.py
│   └── StorageType.AZURE_BLOB 추가
│
├── src/app/domains/cms/file/di/containers.py
│   └── AzureBlobStorageService 팩토리 등록
│
├── kubernetes/base/cms/configmap.yaml
│   └── AZURE_STORAGE_ACCOUNT_NAME, CONTAINER_NAME 추가
│
└── kubernetes/base/secrets/app-secrets.yaml
    └── AZURE_STORAGE_CONNECTION_STRING, ACCOUNT_KEY 추가

신규 파일:
└── src/app/domains/cms/file/service/azure_blob_storage_service.py
    └── AzureBlobStorageService 클래스 (전체 구현)
```

### 6.3 테스트 단계

- [ ] 단일 빌드 테스트
- [ ] 동시 빌드 테스트 (5개, 10개, 20개)
- [ ] 성능 측정 및 비교
- [ ] 장애 시나리오 테스트

### 6.4 배포 단계

- [ ] 기존 데이터 마이그레이션 (PVC → Blob)
- [ ] 새 PVC (캐시 전용) 배포
- [ ] 애플리케이션 배포
- [ ] 기존 PVC 삭제 (데이터 확인 후)

---

## 7. 요약

### 7.1 변경 전후 비교

```
┌─────────────────────────────────────────────────────────────────┐
│                        변경 전 (AS-IS)                          │
├─────────────────────────────────────────────────────────────────┤
│  Storage: Azure Files Premium 100Gi                             │
│  특징: 모든 Pod가 동일 볼륨 공유                                │
│  한계: 동시 빌드 10개 이상 시 심각한 성능 저하                  │
│  비용: $17.50/월                                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        변경 후 (TO-BE)                          │
├─────────────────────────────────────────────────────────────────┤
│  Storage: Azure Blob + PVC 20Gi (캐시)                         │
│  특징: 각 Job이 독립적 로컬 스토리지 사용                       │
│  성능: 동시 빌드 50개도 개별 빌드와 비슷한 시간                 │
│  비용: $10.50/월 (40% 절감)                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 기대 효과

| 지표 | 현재 | 목표 | 개선율 |
|------|-----:|-----:|-------:|
| 최대 동시 빌드 | 10개 | **50개+** | 5배 |
| 동시 빌드 시 성능 저하 | 3-5배 느림 | **거의 없음** | - |
| 월 비용 | $17.50 | $10.50 | **-40%** |
| 스토리지 확장성 | 수동 | **자동** | - |
| 다중 리전 지원 | 불가 | **가능** | - |

### 7.3 권장 우선순위

```
P0 (완료) → P1 (현재) → 스토리지 개선 (추가)

권장 순서:
1. P1 작업 (로깅, KeyVault 등) - 기존 계획
2. 스토리지 개선 - 스케일 목표 달성 필수
3. P2 작업 (ArgoCD, KEDA 등)
```
