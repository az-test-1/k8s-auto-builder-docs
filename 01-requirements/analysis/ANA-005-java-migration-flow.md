# Java 8→17, Spring Boot 2→3 전환 - 전체 실행 흐름

> 유즈케이스: Java 8 → 17, Spring Boot 2 → 3 마이그레이션
> 대상 시스템: auto-builder-cms

---

## 1. 전체 흐름 다이어그램

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           사용자 요청                                        │
│                POST /api/v1/tr/projects/{id}/runs                           │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  TrRunManageHandler.create_and_run()                                        │
│  ├─ TrRunManageEntity 생성 (status: STANDBY)                                │
│  ├─ RunSteps 구성 (프로젝트 설정에서)                                         │
│  ├─ RunProjectSnapshot 생성 (실행 시점 스냅샷)                                │
│  └─ RunStarter.start_run(run_id) → Celery 큐에 전달                          │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  Celery Worker: execute_run_task(run_id)                                    │
│  ├─ Redis Lock 획득 (중복 실행 방지)                                         │
│  └─ RunManager 생성 → start_run()                                           │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  RunManager._execute_run() - 스텝 순차 실행                                  │
│                                                                             │
│  while next_step:                                                           │
│      ├─ StepExecutorFactory.create_executor(step_type)                     │
│      ├─ executor.start(context, settings)                                  │
│      ├─ executor.run()  ← 실제 작업 수행                                    │
│      └─ context.complete_step(success)                                     │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  각 Step에서 컨테이너 실행                                                   │
│                                                                             │
│  ContainerManagerFactory.get_manager()                                      │
│      ↓                                                                      │
│  if CONTAINER_RUNTIME == "kubernetes":                                      │
│      K8sJobManager  ← BUILD_IMAGE_REGISTRY 사용                             │
│  else:                                                                      │
│      DockerManager                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Step 정의 (Java Migration Pipeline)

| 순서 | Step 이름 | 설명 | 실행 내용 |
|:---:|-----------|------|----------|
| 0 | `SETUP_PROJECT` | 프로젝트 설정 | Git clone 또는 ZIP 업로드 |
| 1 | `OPENREWRITE` | 코드 변환 | Java 8→17 문법 변환, Spring Boot 2→3 레시피 적용 |
| 2 | `SQL` | SQL 마이그레이션 | 필요시 SQL 문법 변환 |
| 3 | `BUILD_LLM` | 빌드 에러 수정 | Maven/Gradle 빌드 + LLM 에러 수정 |
| 4 | `FEW_SHOT_GENERATE` | 학습 데이터 생성 | 에러 수정 패턴 기록 |

---

## 3. 핵심 파일별 역할

```
src/app/domains/tr/tr_run/
├── router/
│   └── tr_run_manage_router.py      ← API 엔드포인트 정의
│
├── handler/
│   └── tr_run_manage_handler.py     ← 비즈니스 로직 (TR 생성/실행)
│
├── run/
│   ├── run_celery_task.py           ← Celery 비동기 태스크
│   ├── run_manager.py               ← 스텝 오케스트레이션
│   ├── run_context.py               ← 실행 상태 관리
│   ├── tr_run_step_model.py         ← 스텝 타입 정의
│   ├── step_executor_factory.py     ← Executor 팩토리
│   │
│   ├── step_executors/
│   │   ├── base_step_executor.py    ← 공통 베이스 클래스
│   │   ├── setup_project_executor.py
│   │   ├── openrewrite_executor.py  ← Java 코드 변환
│   │   ├── build_llm_executor.py    ← 빌드 + LLM
│   │   └── python_migration_executor.py
│   │
│   └── utils/
│       ├── container_manager.py     ← Docker/K8s 팩토리
│       ├── docker_manager.py        ← Docker 컨테이너 실행
│       └── k8s_job_manager.py       ← K8s Job 생성/모니터링
```

---

## 4. 단계별 상세 흐름

### Step 0: SETUP_PROJECT

```python
# setup_project_executor.py
async def run(self):
    # 1. Git clone 또는 ZIP 압축 해제
    if project.source_type == "GIT":
        await git_clone(project.git_url, work_dir)
    else:
        await extract_zip(uploaded_file, work_dir)

    # 2. Git 초기화 (이력 관리용)
    await git_init(work_dir)
    await git_commit("Initial commit")
```

### Step 1: OPENREWRITE (Java 8→17)

```python
# openrewrite_executor.py
async def run(self):
    # Sub-step 1: REWRITE_RUN
    # - OpenRewrite 레시피 적용
    # - Java 버전 업그레이드 (8→17)
    # - Spring Boot 버전 업그레이드 (2→3)

    recipe = """
    org.openrewrite.java.migrate.Java8toJava11
    org.openrewrite.java.migrate.JavaVersion17
    org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_0
    """

    # 컨테이너에서 Maven/Gradle 빌드 실행
    await self.container_manager.run({
        "image": "maven:3.8-openjdk-17",  # or from BUILD_IMAGE_REGISTRY
        "command": "mvn rewrite:run",
        "volumes": {project_path: "/workspace"}
    })

    # Sub-step 2: LIB_LLM
    # - LLM으로 라이브러리 의존성 업데이트
    # - pom.xml / build.gradle 수정
```

### Step 3: BUILD_LLM (빌드 에러 수정)

```python
# build_llm_executor.py
async def run(self):
    max_iterations = 10

    for i in range(max_iterations):
        # 1. 컨테이너에서 빌드 실행
        result = await self.container_manager.run({
            "image": self._get_build_image(),  # BUILD_IMAGE_REGISTRY 참조
            "command": "mvn compile",
            "volumes": {project_path: "/workspace"}
        })

        if result.success:
            return True

        # 2. 빌드 에러 분석 (LLM)
        errors = parse_build_errors(result.output)

        # 3. LLM에게 수정 요청
        fixes = await self.llm_manager.fix_compilation_errors(
            source_files=error_files,
            errors=errors
        )

        # 4. 수정 적용
        for fix in fixes:
            await write_file(fix.file_path, fix.content)

    return False  # 최대 반복 후 실패
```

---

## 5. K8s Job 생성 흐름 (BUILD_IMAGE_REGISTRY 사용)

```python
# k8s_job_manager.py
class K8sJobManager:
    async def run(self, config: K8sJobConfig) -> dict:
        settings = get_settings()

        # 1. 이미지 경로 결정
        if settings.BUILD_IMAGE_REGISTRY:
            # ACR 사용: acraz01cloudtr.azurecr.io/docker.io/library/maven:3.8-openjdk-17
            image = f"{settings.BUILD_IMAGE_REGISTRY}/{config.image}"
        else:
            # Docker Hub 직접 사용: maven:3.8-openjdk-17
            image = config.image

        # 2. K8s Job 매니페스트 생성
        job = V1Job(
            metadata=V1ObjectMeta(name=f"tr-build-{uuid}"),
            spec=V1JobSpec(
                template=V1PodTemplateSpec(
                    spec=V1PodSpec(
                        containers=[V1Container(
                            name="build",
                            image=image,  # ← BUILD_IMAGE_REGISTRY 적용됨
                            command=config.command,
                            volumeMounts=[
                                V1VolumeMount(
                                    name="data",
                                    mountPath="/data/cms"  # K8S_DATA_MOUNT_PATH
                                )
                            ],
                            resources=V1ResourceRequirements(
                                limits={"memory": "4Gi", "cpu": "2"},
                                requests={"memory": "1Gi", "cpu": "500m"}
                            )
                        )],
                        volumes=[
                            V1Volume(
                                name="data",
                                persistentVolumeClaim=V1PVCVolumeSource(
                                    claimName="cms-data-pvc"  # K8S_DATA_PVC_NAME
                                )
                            )
                        ],
                        imagePullSecrets=[
                            V1LocalObjectReference(name="acr-secret")
                        ]
                    )
                )
            )
        )

        # 3. Job 제출 및 모니터링
        batch_api.create_namespaced_job(namespace, job)

        # 4. 완료 대기
        while not completed:
            status = batch_api.read_namespaced_job_status(job_name, namespace)
            if status.status.succeeded:
                return {"success": True}
            if status.status.failed:
                logs = self._get_pod_logs(job_name)
                return {"success": False, "error": logs}
```

---

## 6. 설정 참조 관계

```
┌─────────────────────────────────────────────────────────────────┐
│  ConfigMap (cms-config)                                         │
├─────────────────────────────────────────────────────────────────┤
│  BUILD_IMAGE_REGISTRY: "acraz01cloudtr.azurecr.io"              │
│  CONTAINER_RUNTIME: "kubernetes"                                │
│  K8S_NAMESPACE: "dev"                              │
│  K8S_DATA_PVC_NAME: "cms-data-pvc"                              │
│  K8S_DATA_MOUNT_PATH: "/data/cms"                               │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  config.py (Settings)                                           │
│  settings = get_settings()                                      │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
           ┌─────────────────┼─────────────────┐
           ↓                 ↓                 ↓
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ container_manager│ │ k8s_job_manager  │ │ step_executors   │
│ .py              │ │ .py              │ │ /*.py            │
├──────────────────┤ ├──────────────────┤ ├──────────────────┤
│ CONTAINER_RUNTIME│ │ BUILD_IMAGE_REG  │ │ BUILD_IMAGE_REG  │
│ 으로 분기 결정    │ │ 로 이미지 경로   │ │ 로 빌드 이미지   │
│                  │ │ 조합             │ │ 결정             │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

---

## 7. 실제 실행 예시 (Java 8→17)

```
[12:00:00] TR Run #42 시작 (Project: my-spring-app)

[12:00:01] Step 0: SETUP_PROJECT
           └─ Git clone: https://github.com/user/my-spring-app.git
           └─ 완료 (15초)

[12:00:16] Step 1: OPENREWRITE
           ├─ Sub-step: REWRITE_RUN
           │  └─ K8s Job 생성: tr-rewrite-abc123
           │  └─ Image: acraz01cloudtr.azurecr.io/maven:3.8-openjdk-17
           │  └─ Command: mvn rewrite:run
           │  └─ 완료 (3분 20초)
           └─ Sub-step: LIB_LLM
              └─ LLM으로 pom.xml 의존성 업데이트
              └─ 완료 (45초)

[12:04:21] Step 3: BUILD_LLM
           ├─ Iteration 1:
           │  └─ K8s Job 생성: tr-build-def456
           │  └─ Image: acraz01cloudtr.azurecr.io/maven:3.8-openjdk-17
           │  └─ mvn compile → 실패 (12개 에러)
           │  └─ LLM 분석 → 8개 파일 수정
           ├─ Iteration 2:
           │  └─ mvn compile → 실패 (3개 에러)
           │  └─ LLM 분석 → 2개 파일 수정
           └─ Iteration 3:
              └─ mvn compile → 성공!
              └─ 완료 (5분 10초)

[12:09:31] TR Run #42 완료 (status: COMPLETED)
```

---

## 8. 시퀀스 다이어그램

```
User          API           Handler        Celery         RunManager      Executor       K8sJobManager
 │             │              │              │               │               │               │
 │──POST /runs→│              │              │               │               │               │
 │             │──create()───→│              │               │               │               │
 │             │              │──persist────→│               │               │               │
 │             │              │              │               │               │               │
 │             │              │──start_run()→│               │               │               │
 │             │◀─run_id──────│              │               │               │               │
 │◀─202────────│              │              │               │               │               │
 │             │              │              │──task()──────→│               │               │
 │             │              │              │               │──get_next()──→│               │
 │             │              │              │               │               │               │
 │             │              │              │               │──SETUP_PROJECT│               │
 │             │              │              │               │◀──success─────│               │
 │             │              │              │               │               │               │
 │             │              │              │               │──OPENREWRITE──│               │
 │             │              │              │               │               │──run()───────→│
 │             │              │              │               │               │               │──create_job()
 │             │              │              │               │               │               │──monitor()
 │             │              │              │               │               │◀──result──────│
 │             │              │              │               │◀──success─────│               │
 │             │              │              │               │               │               │
 │             │              │              │               │──BUILD_LLM────│               │
 │             │              │              │               │               │──run()───────→│
 │             │              │              │               │               │               │──create_job()
 │             │              │              │               │               │──LLM_fix()    │
 │             │              │              │               │               │──run()───────→│
 │             │              │              │               │               │◀──success─────│
 │             │              │              │               │◀──success─────│               │
 │             │              │              │               │               │               │
 │             │              │              │◀──COMPLETED───│               │               │
 │             │              │              │               │               │               │
```

---

## 9. 구성요소 요약

| 구성요소 | 역할 | BUILD_IMAGE_REGISTRY 사용 |
|----------|------|--------------------------|
| **API Router** | HTTP 요청 수신 | - |
| **Handler** | TR 생성, Celery 트리거 | - |
| **RunManager** | 스텝 순차 실행 | - |
| **StepExecutor** | 각 스텝 로직 구현 | 빌드 이미지 결정 |
| **ContainerManager** | Docker/K8s 분기 | 런타임 선택 |
| **K8sJobManager** | K8s Job 생성/모니터링 | **이미지 prefix로 사용** |

---

## 10. 관련 환경 변수

| 변수명 | 용도 | 예시 값 |
|--------|------|---------|
| `CONTAINER_RUNTIME` | 컨테이너 런타임 선택 | `kubernetes` |
| `BUILD_IMAGE_REGISTRY` | 빌드 이미지 레지스트리 | `acraz01cloudtr.azurecr.io` |
| `K8S_NAMESPACE` | K8s 네임스페이스 | `dev` |
| `K8S_DATA_PVC_NAME` | 데이터 PVC 이름 | `cms-data-pvc` |
| `K8S_DATA_MOUNT_PATH` | 컨테이너 내 마운트 경로 | `/data/cms` |
| `K8S_JOB_MEMORY_LIMIT` | Job 메모리 제한 | `4Gi` |
| `K8S_JOB_CPU_LIMIT` | Job CPU 제한 | `2` |
| `K8S_JOB_ACTIVE_DEADLINE` | Job 타임아웃 (초) | `3600` |
| `K8S_IMAGE_PULL_SECRETS` | 이미지 풀 시크릿 | `acr-secret` |

---

## 11. 에러 처리 흐름

```
Step 실행 중 에러 발생
    ↓
┌─────────────────────────────────────────┐
│ K8s Job 실패 시                          │
│ ├─ Pod 로그 수집                         │
│ ├─ 에러 메시지 파싱                       │
│ └─ RunContext에 에러 기록                 │
└────────────────────┬────────────────────┘
                     ↓
┌─────────────────────────────────────────┐
│ BUILD_LLM 단계에서                       │
│ ├─ LLM으로 에러 분석                     │
│ ├─ 코드 수정 제안                        │
│ ├─ 자동 적용 후 재빌드                    │
│ └─ 최대 반복 횟수까지 시도                 │
└────────────────────┬────────────────────┘
                     ↓
┌─────────────────────────────────────────┐
│ 최종 실패 시                             │
│ ├─ Step status: FAILED                  │
│ ├─ Run status: FAILED                   │
│ └─ 에러 로그 저장                        │
└─────────────────────────────────────────┘
```

---

## 12. 참고 문서

- [ANA-002-cicd-kustomize-roles](./ANA-002-cicd-kustomize-roles.md)
- [ANA-006-env-variables](./ANA-006-env-variables.md)
