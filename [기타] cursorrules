# Auto Builder CMS (Dev) - Cursor AI Context & Rules

## CRITICAL: 기존 구조 절대 변경 금지!
- 파일/폴더 구조 바꾸지 마
- 리팩토링 하지 마

---

# PART 1: 프로젝트 개요

## 1.1 제품 소개
**Auto Builder CMS** = 레거시 Java 프로젝트를 최신 버전으로 자동 마이그레이션하는 플랫폼

## 1.2 레포지토리 구조
```
dev-refact/
├── auto-builder-cms-dev/    # Backend (FastAPI)
│   └── src/app/
│       ├── common/          # 공통 모듈
│       ├── config/          # 설정
│       ├── core/            # 핵심 모듈
│       ├── domains/         # 도메인별 비즈니스 로직
│       │   ├── cms/         # CMS 관련 (admin, login, menu 등)
│       │   └── tr/          # 마이그레이션 관련 (핵심)
│       └── entrypoint/      # 진입점
└── auto-builder-ui-dev/     # Frontend (Next.js)
    └── src/
        ├── api/             # API 통신
        ├── app/             # Next.js App Router
        ├── layouts/         # 레이아웃 컴포넌트
        ├── libs/            # 라이브러리
        ├── page/            # 페이지 컴포넌트
        └── utils/           # 유틸리티
```

## 1.3 핵심 기능
1. **그룹 관리**: 마이그레이션 설정 템플릿
2. **프로젝트 관리**: 마이그레이션 대상 프로젝트
3. **프롬프트 관리**: LLM 호출용 프롬프트
4. **태그 관리**: 프로젝트 분류
5. **실행 관리**: 마이그레이션 실행/모니터링
6. **퓨샷 관리**: LLM 수정 예시 관리 (AS-IS/TO-BE)

## 1.4 기술 스택
- Backend: FastAPI, SQLAlchemy Async, Celery, LangChain, LangGraph, LangSmith
- Frontend: Next.js 15, React 19, JSX
- DB: PostgreSQL
- Container: Docker (DooD 방식)
- LLM: OpenAI, Claude (Anthropic)

---

# PART 2: 필수 입력값 (화면별)

## 2.1 그룹 (Group) - 2개 필수

| 필드명 | 필수 | 설명 | 타입 |
|--------|:----:|------|------|
| `group_name` | ✅ | 그룹명 | string (1~255) |
| `group_code` | ✅ | 그룹코드 | string (unique) |
| `project_language_version` | | 소스 언어 버전 | string |
| `target_language` | | 타겟 언어 | enum |
| `target_language_version` | | 타겟 언어 버전 | string |
| `target_build_tool` | | 타겟 빌드도구 | enum |
| `openrewrite_recipe` | | OpenRewrite 레시피 | text |
| `openrewrite_recipe_active_name` | | 활성 레시피명 | string |
| `build_prompt_codes` | | 빌드 프롬프트 코드 | list |
| `use_yn` | | 사용여부 | string (기본값 "Y") |

## 2.2 프로젝트 (Project) - 3개 필수 (+조건부)

| 필드명 | 필수 | 설명 | 타입 |
|--------|:----:|------|------|
| `project_name` | ✅ | 프로젝트명 | string (1~255) |
| `project_code` | ✅ | 프로젝트코드 | string (unique) |
| `upload_type` | ✅ | 업로드 타입 | enum (GIT, FILE) |
| `git_url` | 조건 | Git URL | string (GIT일때 필수) |
| `project_language` | | 소스 언어 | enum |
| `project_build_tool` | | 소스 빌드도구 | enum |
| `target_language_version` | | 타겟 언어 버전 | string |
| 기타... | | | |

## 2.3 프롬프트 (Prompt) - 3개 필수

| 필드명 | 필수 | 설명 | 타입 |
|--------|:----:|------|------|
| `prompt_name` | ✅ | 프롬프트명 | string (1~255) |
| `prompt_code` | ✅ | 프롬프트코드 | string |
| `prompt` | ✅ | 프롬프트 내용 | text |
| `llm_model` | | LLM 모델 | enum (기본값 GPT4O) |
| `prompt_type` | | 프롬프트 타입 | enum |
| `priority` | | 우선순위 | int (기본값 0) |

## 2.4 태그 (Tag) - 1개 필수

| 필드명 | 필수 | 설명 | 타입 |
|--------|:----:|------|------|
| `tag_name` | ✅ | 태그명 | string (unique) |

## 2.5 퓨샷 (Fewshot) - 3개 필수

| 필드명 | 필수 | 설명 | 타입 |
|--------|:----:|------|------|
| `fewshot_name` | ✅ | 퓨샷명 | string (1~255) |
| `as_is` | ✅ | AS-IS 코드 | text |
| `to_be` | ✅ | TO-BE 코드 | text |
| `llm_tool_description` | | LLM 툴 설명 | text |
| `llm_use_count` | | LLM 사용 횟수 | int (기본값 0) |
| `use_yn` | | 사용여부 | string (기본값 "Y") |
| `fewshot_status` | | 퓨샷 상태 | enum (기본값 REQUIRED) |
| `tag_ids` | | 태그 ID 리스트 | list[int] |

---

# PART 3: Pydantic 필드 규칙 (중요!)

## 3.1 필수/선택 필드 작성법

```python
# ✅ 올바른 방법: 필수 필드
field_name: Annotated[str, Field(description="설명")]

# ✅ 올바른 방법: 선택 필드 (Optional + default)
field_name: Annotated[Optional[str], Field(default=None, description="설명")]

# ❌ 잘못된 방법: 타입은 str인데 default=None (Pydantic 에러!)
field_name: Annotated[str, Field(default=None, description="설명")]
```

## 3.2 Backend 수정 시 규칙

```python
# Request 모델에서 선택 필드는 반드시 이렇게:
from typing import Optional, Annotated
from pydantic import Field

class CreateReqModel(CustomBaseModel):
    # 필수
    name: Annotated[str, Field(description="이름", min_length=1)]
    code: Annotated[str, Field(description="코드", min_length=1)]

    # 선택 (Optional + default 필수!)
    description: Annotated[Optional[str], Field(default=None, description="설명")]
    version: Annotated[Optional[str], Field(default=None, description="버전")]
    count: Annotated[Optional[int], Field(default=0, description="횟수")]
    items: Annotated[Optional[List[str]], Field(default_factory=list, description="목록")]
    use_yn: Annotated[Optional[str], Field(default="Y", description="사용여부")]
```

## 3.3 Frontend 수정 시 규칙

```javascript
// 필수 필드 체크
const requiredFields = {
    group: ['group_name', 'group_code'],
    project: ['project_name', 'project_code', 'upload_type'],
    prompt: ['prompt_name', 'prompt_code', 'prompt']
}

// 빈값 허용 (선택 필드)
const optionalFields = {
    group: ['target_language', 'target_language_version', 'openrewrite_recipe', ...],
    project: ['git_url', 'project_language', ...],
    prompt: ['llm_model', 'priority', ...]
}

// API 전송 시 빈값은 null 또는 제외
const cleanData = (data) => {
    return Object.fromEntries(
        Object.entries(data).filter(([k, v]) => v !== '' && v !== undefined)
    )
}
```

## 3.4 DB 컬럼 규칙

```sql
-- 필수 컬럼: NOT NULL
group_name VARCHAR(255) NOT NULL,
group_code VARCHAR(255) NOT NULL UNIQUE,

-- 선택 컬럼: NULL 허용 (NOT NULL 없음)
target_language VARCHAR(50),
target_language_version VARCHAR(255),
openrewrite_recipe TEXT,
```

---

# PART 4: Enum 값 (전체)

## 4.1 UploadType (업로드 타입)
```python
class UploadType(BaseEnum):
    FILE = "파일업로드"
    GIT = "git"
```

## 4.2 ProgramLanguage (프로그래밍 언어)
```python
class ProgramLanguage(BaseEnum):
    JAVA = "java"           # 버전: 21, 17, 11, 8, 7, 6
    PYTHON = "python"       # 버전: 3.12, 3.11, 3.10, 3.9, 3.8, 3.7, 3.6, 2.7
    JAVASCRIPT = "javascript"  # 버전: 20, 18, 16, 14
    TYPESCRIPT = "typescript"  # 버전: 5.3, 5.0, 4.9, 4.5, 4.0
    KOTLIN = "kotlin"       # 버전: 1.9, 1.8, 1.7, 1.6
    GO = "go"               # 버전: 1.21, 1.20, 1.19, 1.18, 1.17
    RUST = "rust"           # 버전: 1.74, 1.73, 1.72, 1.71, 1.70
    PHP = "php"             # 버전: 8.3, 8.2, 8.1, 8.0, 7.4
    RUBY = "ruby"           # 버전: 3.3, 3.2, 3.1, 3.0, 2.7
    CSHARP = "csharp"       # 버전: 12, 11, 10, 9, 8
    CPP = "cpp"             # 버전: 23, 20, 17, 14, 11
```

## 4.3 BuildTool (빌드 도구)
```python
class BuildTool(BaseEnumWithFilters):
    # Java/Kotlin
    GRADLE = "gradle"       # 버전: 8.5, 8.0, 7.6, 7.0, 6.9
    MAVEN = "maven"         # 버전: 3.9.6, 3.9.0, 3.8.8, 3.6.3
    # JavaScript/TypeScript
    NPM = "npm"             # 버전: 10.2.0, 9.6.0, 8.19.0, 6.14.0
    YARN = "yarn"           # 버전: 4.0.0, 3.6.0, 1.22.0
    PNPM = "pnpm"           # 버전: 8.0.0, 7.0.0, 6.0.0
    # Python
    PIP = "pip"             # 버전: 23.3, 23.0, 22.3, 21.3, 20.3
    POETRY = "poetry"       # 버전: 1.7.0, 1.6.0, 1.5.0, 1.4.0
    PIPENV = "pipenv"       # 버전: 2023.11.0, 2023.10.0, 2023.9.0
    # Others
    GO_MOD = "go_mod"
    CARGO = "cargo"
    COMPOSER = "composer"
    BUNDLER = "bundler"
    DOTNET = "dotnet"
    CMAKE = "cmake"
    MAKE = "make"
```

## 4.4 PromptType (프롬프트 타입)
```python
class PromptType(BaseEnum):
    LIB = "Dependency"      # 라이브러리/의존성
    SQL = "SQL"             # SQL 변환
    BUILD = "SpringBoot"    # 빌드 에러 수정
```

## 4.5 LlmModel (LLM 모델)
```python
class LlmModel(BaseEnum):
    OPENAI = "OpenAI"
    CLAUDE = "Claude"
```

## 4.6 RunStatus (실행 상태)
```python
class RunStatus(BaseEnum):
    STANDBY = "Standby"
    RUNNING = "Running"
    CANCELLED = "Canceled"
    COMPLETED = "Completed"
    FAILED = "Failed"
```

## 4.7 StepStatus (스텝 상태)
```python
class StepStatus(BaseEnum):
    STANDBY = "standby"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    SKIPPED = "skipped"
```

## 4.8 MainStepType (메인 스텝)
```python
class MainStepType(BaseEnum):
    SETUP_PROJECT = "SetupProject"    # 0: 프로젝트 준비 (Git/Upload)
    OPENREWRITE = "Openrewrite"       # 1: 소스코드 업그레이드
    SQL = "SQL"                       # 2: SQL 변환
    BUILD_LLM = "Build+LLM"           # 3: 빌드 에러 LLM 수정
    BUILD_LLM_MCP = "Build+LLM_MCP"   # 3: 빌드 에러 MCP 수정
    FEW_SHOT_GENERATE = "FewShot_Generate"  # 4: 퓨샷 히스토리 기록
```

## 4.9 FewShotStatus (퓨샷 상태)
```python
class FewShotStatus(BaseEnum):
    REQUIRED = "검증필요"
    COMPLETE = "검증완료"
```

## 4.10 언어별 빌드도구 매핑
```
JAVA/KOTLIN  → GRADLE, MAVEN
PYTHON       → PIP, POETRY, PIPENV
JAVASCRIPT/TYPESCRIPT → NPM, YARN, PNPM
GO           → GO_MOD
RUST         → CARGO
PHP          → COMPOSER
RUBY         → BUNDLER
CSHARP       → DOTNET
CPP          → CMAKE, MAKE
```

---

# PART 5: DB 스키마

## 5.1 tbl_tr_group
```
id (PK), group_name (NOT NULL), group_code (NOT NULL UNIQUE),
target_language, project_language_version, target_language_version,
target_build_tool, openrewrite_recipe, openrewrite_recipe_active_name,
build_prompt_codes, use_yn, delete_yn,
created_at, updated_at, created_by, updated_by, created_from, updated_from
```

## 5.2 tbl_tr_project
```
id (PK), project_name (NOT NULL), project_code (NOT NULL UNIQUE),
tr_group_id (FK), upload_type (NOT NULL),
git_url, git_username, git_password, git_token, git_branch,
file_upload_id_pk, project_language, project_language_version,
project_build_tool, project_build_tool_version, target_language, target_language_version,
target_build_tool, target_build_tool_version, openrewrite_recipe, openrewrite_recipe_active_name,
lib_prompt_code, build_max_count, build_file_max_count, build_prompt_codes,
sql_prompt_code, step_names, delete_yn,
created_at, updated_at, created_by, updated_by, created_from, updated_from
```

## 5.3 tbl_tr_prompt_manage
```
id (PK), prompt_name (NOT NULL), prompt_code (NOT NULL),
version, prompt (NOT NULL), llm_model, priority, prompt_type, use_yn, delete_yn,
created_at, updated_at, created_by, updated_by, created_from, updated_from
```

## 5.4 tbl_tr_tag
```
id (PK), tag_name (NOT NULL UNIQUE), tag_description, use_yn, delete_yn,
created_at, updated_at, created_by, updated_by, created_from, updated_from
```

## 5.5 tbl_tr_fewshot_manage
```
id (PK), fewshot_name (NOT NULL), llm_tool_description, llm_use_count (기본값 0),
as_is (NOT NULL), to_be (NOT NULL), use_yn (기본값 "Y"),
fewshot_status (기본값 "검증필요"), delete_yn,
created_at, updated_at, created_by, updated_by, created_from, updated_from
```

## 5.6 tbl_tr_fewshot_tag_rel (다대다 관계 테이블)
```
id (PK), fewshot_id (FK → tbl_tr_fewshot_manage), tag_id (FK → tbl_tr_tag),
UNIQUE(fewshot_id, tag_id),
created_at, updated_at, created_by, updated_by, created_from, updated_from
```

---

# PART 6: Backend 구조 (8 레이어 유지!)

```
domains/tr/{entity}/
├── di/containers.py
├── entity/{entity}.py
├── handler/{entity}_handler.py
├── mapper/{entity}_mapper.py
├── model/
│   ├── {entity}_model.py
│   ├── {entity}_request.py
│   └── {entity}_response.py
├── repository/{entity}_repository.py
├── router/{entity}_router.py
└── service/{entity}_service.py
```

---

# PART 7: API 엔드포인트

```
# 그룹
GET/POST   /api/v1/tr/group
GET/PUT/DELETE /api/v1/tr/group/{id}

# 프로젝트
GET/POST   /api/v1/tr/project
GET/PUT/DELETE /api/v1/tr/project/{id}

# 프롬프트
GET/POST   /api/v1/tr/prompt
GET/PUT/DELETE /api/v1/tr/prompt/{id}

# 태그
GET/POST   /api/v1/tr/tag
DELETE     /api/v1/tr/tag/{id}

# 퓨샷
GET/POST   /api/v1/tr/fewshots
GET/PUT/DELETE /api/v1/tr/fewshots/{id}
GET        /api/v1/tr/fewshots/export/excel  # 엑셀 내보내기
```

---

# PART 8: 개발 규칙

## 절대 하지 말 것
1. ❌ 폴더/파일 구조 변경
2. ❌ 레이어 추가/삭제
3. ❌ JSX → TSX 변환
4. ❌ 리팩토링/최적화 제안
5. ❌ "더 나은 방법" 제안
6. ❌ 쓸데없는 디버깅 로그 추가 (console.log, print 등)

## 반드시 할 것
1. ✅ 기존 패턴 그대로 복사해서 수정
2. ✅ 기존 import 경로 따르기
3. ✅ 요청한 부분만 변경
4. ✅ 기존 코드 스타일 유지
5. ✅ Optional 필드는 반드시 `Optional[타입]` + `default` 사용

## 새 기능 추가 시
```bash
# 비슷한 기존 폴더 복사 → 이름만 변경 → 필요한 부분만 수정
cp -r domains/tr/tr_project domains/tr/tr_new_feature
```

---

# PART 9: 필수값 요약

| 화면 | 필수 개수 | 필수 필드 |
|------|:---------:|-----------|
| 그룹 | **2개** | group_name, group_code |
| 프로젝트 | **3개** | project_name, project_code, upload_type |
| 프롬프트 | **3개** | prompt_name, prompt_code, prompt |
| 태그 | **1개** | tag_name |
| 퓨샷 | **3개** | fewshot_name, as_is, to_be |

---

# PART 10: API 통신 규칙 (CORS & fetchUtils)

## 10.1 CORS 설정 (Backend)

```python
# main.py - CORS 미들웨어 설정 (절대 변경 금지!)
from fastapi.middleware.cors import CORSMiddleware

_app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.BACKEND_CORS_ORIGINS,  # ["*"] 또는 도메인 목록
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=[
        "Content-Disposition",
        "Content-Length",
        "Cache-Control",
        "Content-Type",
    ],
)

# config.py
BACKEND_CORS_ORIGINS: Union[str, List[str]] = ["*"]
```

## 10.2 fetchUtils 사용법 (Frontend)

```javascript
// src/api/fetch/fetchUtils.js
import { getFetch, postFetch, putFetch, deleteFetch, uploadFetch } from '@/api/fetch/fetchUtils'

// GET 요청
const data = await getFetch('/api/v1/tr/group', queryParams, pathParams, options)

// POST 요청 (body 자동 snake_case 변환)
const result = await postFetch('/api/v1/tr/group', body, queryParams, pathParams, options)

// PUT 요청
const result = await putFetch('/api/v1/tr/group', body, queryParams, [id], options)

// DELETE 요청
const result = await deleteFetch('/api/v1/tr/group', queryParams, [id], options)

// 파일 업로드
const result = await uploadFetch('/api/v1/file/upload', formData, queryParams, pathParams, options)
```

## 10.3 Case 변환 규칙 (중요!)

```
Frontend (JS)          ←→          Backend (Python)
  camelCase                          snake_case

  projectName         POST→         project_name
  targetVersion       요청시         target_version

  projectName         ←GET          project_name
  targetVersion       응답시         target_version
```

**자동 변환 로직:**
```javascript
// src/utils/caseConverter.js (절대 수정 금지!)

// 요청 시: camelCase → snake_case
const transformedBody = toSnakeCase(body)

// 응답 시: snake_case → camelCase
const transformedData = toCamelCase(data)
```

## 10.4 fetchUtils 핵심 함수 (수정 금지!)

```javascript
// 1. normalizeBaseUrl - 환경별 API URL 결정
const normalizeBaseUrl = (baseUrl, options) => {
  // 개발/운영 도메인: window.location.origin 사용
  // localhost: NEXT_PUBLIC_SERVER_URL 환경변수 사용
}

// 2. buildUrl - URL 조합
const buildUrl = (path, queryParams, pathParams, options) => {
  // baseUrl + path + pathParams + queryParams
}

// 3. setHeaders - 헤더 설정
const setHeaders = async (contentType = 'json') => {
  // Content-Type 설정
  // Authorization: Bearer 토큰 추가
}

// 4. handleResponse - 응답 처리
const handleResponse = async (response, isBlob, rawResponse) => {
  // 401 에러 → 로그아웃
  // JSON 파싱 + snake_case → camelCase 변환
}
```

## 10.5 환경 변수

```bash
# Frontend (.env)
NEXT_PUBLIC_SERVER_URL=http://localhost:18001  # 로컬 개발용

# Backend (.env)
BACKEND_CORS_ORIGINS=["*"]  # 또는 ["https://domain.com"]
```

## 10.6 API 요청 흐름

```
[Frontend]                              [Backend]
    │                                       │
    ├─ camelCase body                       │
    │    ↓                                  │
    ├─ toSnakeCase() 변환                   │
    │    ↓                                  │
    ├─ fetch() 요청 ──────────────────────► │
    │                                       ├─ snake_case로 받음
    │                                       ├─ 처리
    │                                       ├─ snake_case로 응답
    │ ◄────────────────────────── 응답 ────┤
    │    ↓                                  │
    ├─ toCamelCase() 변환                   │
    │    ↓                                  │
    └─ camelCase로 사용                     │
```

## 10.7 절대 하지 말 것

1. ❌ fetchUtils.js 수정
2. ❌ caseConverter.js 수정
3. ❌ CORS 설정 변경
4. ❌ 수동으로 snake_case ↔ camelCase 변환
5. ❌ axios 등 다른 HTTP 클라이언트 사용
6. ❌ fetch() 직접 호출 (반드시 fetchUtils 사용)
7. ❌ 쓸데없는 디버깅 로그 추가 (console.log, print, logger.debug 등)

## 10.8 올바른 API 호출 예시

```javascript
// ✅ 올바른 방법
import { getFetch, postFetch } from '@/api/fetch/fetchUtils'

// 목록 조회
const response = await getFetch('/api/v1/tr/group', { page: 1, size: 10 })
const groups = response.data  // 이미 camelCase로 변환됨

// 등록 (camelCase로 작성 → 자동 snake_case 변환)
const result = await postFetch('/api/v1/tr/group', {
  groupName: '그룹명',      // → group_name
  groupCode: 'GROUP_001',   // → group_code
  targetLanguage: 'JAVA'    // → target_language
})

// 수정
const result = await putFetch('/api/v1/tr/group', updateData, null, [id])

// 삭제
const result = await deleteFetch('/api/v1/tr/group', null, [id])
```

---

# PART 11: 언어별 버전 필드 처리 규칙 (중요!)

## 11.1 전체 요약표

| 구분 | Java | Python | PHP | JavaScript |
|:---|:---:|:---:|:---:|:---:|
| **빌드 도구** | Gradle, Maven | PIP, Poetry, Pipenv | Composer | NPM, Yarn, PNPM |
| **언어 버전** | ⚠️ **필수 선택** | 선택 가능 or `n/a` | 선택 가능 or `n/a` | 선택 가능 or `n/a` |
| **빌드도구 버전** | ⚠️ **필수 선택** | `n/a` (자동) | `n/a` (자동) | `n/a` (자동) |
| **타겟 언어 버전** | ⚠️ **필수 선택** | 선택 가능 or `n/a` | 선택 가능 or `n/a` | 선택 가능 or `n/a` |
| **타겟 빌드도구 버전** | ⚠️ **필수 선택** | `n/a` (자동) | `n/a` (자동) | `n/a` (자동) |

## 11.2 Java 시나리오 (모든 버전 필수)

```
┌─────────────────────────────────────────────────────────────────────┐
│  🟢 Java - 모든 버전 필드 필수 선택 (Docker 컨테이너 연동)             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  [소스 설정] - 모두 표시, 모두 필수                                   │
│  ├─ project_language           : java          ← 필수               │
│  ├─ project_language_version   : [Select ▼]    ← ⚠️ 필수 선택       │
│  ├─ project_build_tool         : [Select ▼]    ← ⚠️ 필수 선택       │
│  └─ project_build_tool_version : [Select ▼]    ← ⚠️ 필수 선택       │
│                                                                     │
│  [타겟 설정] - 모두 표시, 모두 필수                                   │
│  ├─ target_language_version    : [Select ▼]    ← ⚠️ 필수 선택       │
│  └─ target_build_tool_version  : [Select ▼]    ← ⚠️ 필수 선택       │
│                                                                     │
│  💾 저장: 모든 버전 필드 반드시 선택 (미선택 시 저장 불가)              │
│  📝 이유: OpenRewrite, Build LLM에서 Docker 컨테이너 코드 필요         │
│          예: JAVA17-MAVEN3.9.6                                       │
└─────────────────────────────────────────────────────────────────────┘
```

## 11.3 Python/PHP/JavaScript 시나리오 (빌드도구 버전 = n/a)

```
┌─────────────────────────────────────────────────────────────────────┐
│  🐍 Python / 🐘 PHP / 🟨 JavaScript - 빌드도구 버전은 자동 "n/a"       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  [소스 설정] - 모두 표시                                              │
│  ├─ project_language           : python/php/js ← 필수               │
│  ├─ project_language_version   : [Select ▼]    ← 선택 가능          │
│  ├─ project_build_tool         : [Select ▼]    ← 선택 가능          │
│  └─ project_build_tool_version : [n/a 표시]    ← 🔒 비활성화        │
│                                                                     │
│  [타겟 설정] - 모두 표시                                              │
│  ├─ target_language_version    : [Select ▼]    ← 선택 가능          │
│  └─ target_build_tool_version  : [n/a 표시]    ← 🔒 비활성화        │
│                                                                     │
│  💾 저장: 언어버전 미선택→"n/a", 빌드도구버전 항상→"n/a"               │
│  📝 이유: OpenRewrite/Build LLM 미사용으로 버전 무관                  │
└─────────────────────────────────────────────────────────────────────┘
```

## 11.4 필드별 처리 규칙

| 필드명 | Java | Python/PHP/JS | 미입력 시 |
|:---|:---:|:---:|:---:|
| `project_language_version` | ⚠️ **필수 선택** | 선택 가능 | `"n/a"` |
| `project_build_tool_version` | ⚠️ **필수 선택** | 🔒 **비활성화** | `"n/a"` |
| `target_language_version` | ⚠️ **필수 선택** | 선택 가능 | `"n/a"` |
| `target_build_tool_version` | ⚠️ **필수 선택** | 🔒 **비활성화** | `"n/a"` |

## 11.5 UI 동작 방식

```
[Java 선택 시]
┌─────────────────────────────────────────────┐
│  언어 버전:     [Select ▼] *필수            ← 활성화
│  빌드도구:      [Select ▼] *필수            ← 활성화
│  빌드도구 버전: [Select ▼] *필수            ← 활성화
└─────────────────────────────────────────────┘

[Python/PHP/JavaScript 선택 시]
┌─────────────────────────────────────────────┐
│  언어 버전:     [Select ▼]                  ← 활성화 (선택적)
│  빌드도구:      [Select ▼]                  ← 활성화 (선택적)
│  빌드도구 버전: [ n/a     ] 🔒              ← 비활성화 (회색)
└─────────────────────────────────────────────┘
```

## 11.6 전송 데이터 예시

```javascript
// Java 선택 시 - 모든 버전 필수
{
  projectLanguage: "java",
  projectLanguageVersion: "8",           // ⚠️ 필수
  projectBuildTool: "maven",
  projectBuildToolVersion: "3.6.3",      // ⚠️ 필수
  targetLanguageVersion: "17",           // ⚠️ 필수
  targetBuildToolVersion: "3.9.6"        // ⚠️ 필수
}

// Python/PHP/JavaScript 선택 시 - 빌드도구 버전은 항상 "n/a"
{
  projectLanguage: "python",             // 또는 "php", "javascript"
  projectLanguageVersion: "3.8",         // 선택값 or "n/a"
  projectBuildTool: "pip",
  projectBuildToolVersion: "n/a",        // 🔒 항상 "n/a"
  targetLanguageVersion: "3.12",         // 선택값 or "n/a"
  targetBuildToolVersion: "n/a"          // 🔒 항상 "n/a"
}
```

## 11.7 Java에서 "n/a" 불가 이유

```python
# build_llm_executor.py:367, openrewrite_executor.py:553
container_code = f"{project_language.name}{target_language_version}-{project_build_tool.name}{target_build_tool_version}"

# 정상 케이스
"JAVA17-MAVEN3.9.6"  ← Docker 컨테이너 찾기 성공 ✅

# n/a 케이스
"JAVA17-MAVENn/a"    ← Docker 컨테이너 찾기 실패 ❌
```

## 11.8 수정 대상 파일

| 위치 | 파일 | 수정 내용 |
|:---|:---|:---|
| **Backend** | `tr_project_request.py` | 빈값 → `"n/a"` 변환 로직 추가 |
| **Frontend** | `ProjectRegisterForm.jsx` | Java: 필수 검증, Python/PHP/JS: 비활성화 UI + `"n/a"` 자동 설정 |

---

# PART 12: 환경 설정

## 12.1 환경 변수

```bash
# Frontend (.env)
NEXT_PUBLIC_SERVER_URL=http://localhost:18001  # 로컬 개발용
NEXT_PUBLIC_MOCK_MODE=false                    # Mock 모드 (MSW)

# Backend (.env)
BACKEND_CORS_ORIGINS=["*"]  # 또는 ["https://domain.com"]
```

## 12.2 포트 설정

| 서비스 | 포트 |
|--------|------|
| Frontend (Next.js) | 15001 |
| Backend (FastAPI) | 18001 |
| MCP Server | 18002 |
| PostgreSQL | 5432 |
| Redis | 6379 |

## 12.3 Node 버전
```bash
node: 22.6.0
```

---

# REMEMBER
- 필수값은 최소화! (name, code 위주)
- 선택 필드는 `Optional[타입]` + `default` 필수
- 기존 코드 패턴 그대로 따라 작업
- **fetchUtils.js, caseConverter.js, CORS 설정 절대 수정 금지!**
- API 호출 시 반드시 fetchUtils 함수 사용
- **Java: 모든 버전 필드 필수, Python/PHP/JS: 빌드도구 버전은 "n/a"**
- **퓨샷 필수값: fewshot_name, as_is, to_be (태그는 tag_ids로 연결)**
