# ADR-M010: 빌드 Job 실행 방식

> **Status**: 승인됨
> **Date**: 2025-12
> **Category**: Migration - Build

---

## 상태
✅ **승인됨** (2025-12)

## 컨텍스트
Auto-Builder는 사용자 프로젝트를 빌드하기 위해 동적으로 컨테이너를 생성해야 함. 기존에는 Docker-in-Docker(DinD)를 사용했으나, K8s 환경에서의 대안 필요

## 고려한 옵션

| 옵션 | 장점 | 단점 |
|------|------|------|
| **A. Docker-in-Docker (DinD)** | 기존 코드 재사용 | 보안 위험, 권한 문제 |
| **B. Docker Socket 마운트** | 단순 | 심각한 보안 위험 |
| **C. Kubernetes Jobs** | K8s 네이티브, 보안 | 코드 변경 필요 |
| **D. Kaniko** | 이미지 빌드 전용 | 범용 빌드에 부적합 |

## 결정
**옵션 C: Kubernetes Jobs**

## 근거
1. **보안**: DinD/Socket 마운트의 보안 위험 제거
2. **K8s 네이티브**: 리소스 관리, 스케줄링 활용
3. **격리**: 각 빌드가 독립된 Pod에서 실행
4. **모니터링**: K8s 표준 도구로 Job 상태 모니터링

## 구현 방식
```python
# CONTAINER_RUNTIME 환경변수로 분기
if settings.CONTAINER_RUNTIME == "kubernetes":
    manager = K8sJobManager()
else:
    manager = DockerManager()
```

## K8s Job 예시
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: tr-build-{uuid}
spec:
  activeDeadlineSeconds: 3600
  template:
    spec:
      containers:
        - name: build
          image: acraz01cloudtr.azurecr.io/java17-maven3.9.6:latest
          command: ["mvn", "compile"]
          volumeMounts:
            - name: data
              mountPath: /data/cms
          resources:
            limits:
              memory: "4Gi"
              cpu: "2"
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: cms-data-pvc
      imagePullSecrets:
        - name: acr-secret
      restartPolicy: Never
```

## 환경 변수
```
CONTAINER_RUNTIME=kubernetes
BUILD_IMAGE_REGISTRY=acraz01cloudtr.azurecr.io
K8S_NAMESPACE=dev
K8S_DATA_PVC_NAME=cms-data-pvc
K8S_DATA_MOUNT_PATH=/data/cms
K8S_JOB_MEMORY_LIMIT=4Gi
K8S_JOB_CPU_LIMIT=2
K8S_JOB_ACTIVE_DEADLINE=3600
```

## 관련 파일
```
src/app/domains/tr/tr_run/run/utils/k8s_job_manager.py
src/app/domains/tr/tr_run/run/utils/container_manager.py
kubernetes/base/cms/configmap.yaml
```

---

*원본: [ADR] Docker-to-Kubernetes-마이그레이션-결정기록.md*
