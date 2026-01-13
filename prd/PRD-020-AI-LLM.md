# PRD-020: AI/LLM

> **Status**: Draft
> **Version**: 1.0.0
> **Last Updated**: 2026-01-13
> **Owner**: AI Team
> **Parent**: [PRD-000-MASTER](./PRD-000-MASTER.md)

---

## 1. 개요

### 1.1 목적
Auto-Builder의 AI/LLM 관련 기능을 정의합니다.

### 1.2 범위
- LLM 기반 빌드 에러 분석 및 수정
- 프롬프트 템플릿 관리
- Few-shot 예시 데이터 관리
- MCP (Model Context Protocol) 연동

---

## 2. 기능 요구사항

### 2.1 LLM 빌드 에러 수정

#### 2.1.1 에러 분석 흐름

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 빌드 실패   │ ──→ │ 에러 로그   │ ──→ │ LLM 분석    │
│ (Maven/     │     │ 파싱        │     │             │
│  Gradle)    │     │             │     │             │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                                               ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 빌드 재실행 │ ←── │ 코드 패치   │ ←── │ 수정 제안   │
│             │     │ 적용        │     │ 생성        │
└─────────────┘     └─────────────┘     └─────────────┘
```

#### 2.1.2 지원 에러 유형
| 에러 유형 | 설명 | 자동 수정률 |
|----------|------|:-----------:|
| 컴파일 에러 | 문법 오류, 타입 불일치 | 높음 |
| 의존성 에러 | 버전 호환성, 누락된 의존성 | 중간 |
| Deprecated API | 폐기된 API 사용 | 높음 |
| 설정 에러 | application.yml, pom.xml 오류 | 중간 |

### 2.2 AI 프롬프트 관리 (F5)

| ID | 기능 | 설명 | 우선순위 |
|----|------|------|:--------:|
| F5.1 | 프롬프트 템플릿 | LLM 프롬프트 커스터마이징 | P2 |
| F5.2 | Few-shot 관리 | 에러 수정 예시 데이터 관리 | P2 |
| F5.3 | 태그 시스템 | 프롬프트/프로젝트 분류 태그 | P2 |

#### 프롬프트 템플릿 구조
```yaml
name: "빌드 에러 수정"
description: "Maven/Gradle 빌드 에러를 분석하고 수정 코드 제안"
system_prompt: |
  당신은 Java 마이그레이션 전문가입니다.
  Java 8에서 17로 마이그레이션 시 발생하는 빌드 에러를 분석하고
  수정 코드를 제안합니다.

user_prompt_template: |
  ## 빌드 에러 로그
  {error_log}

  ## 원본 코드
  {source_code}

  ## 요청
  위 에러를 수정하는 코드 패치를 제안해주세요.

variables:
  - error_log
  - source_code
```

### 2.3 Few-shot 관리

#### Few-shot 데이터 모델
| 필드 | 타입 | 설명 |
|------|------|------|
| id | UUID | 고유 ID |
| error_type | String | 에러 유형 |
| error_pattern | Text | 에러 패턴 (정규식 또는 키워드) |
| original_code | Text | 원본 코드 |
| fixed_code | Text | 수정된 코드 |
| explanation | Text | 수정 설명 |
| tags | Array | 분류 태그 |
| usage_count | Integer | 사용 횟수 |
| success_rate | Float | 성공률 |

#### Few-shot 자동 수집
```
실행 성공 시:
1. 빌드 에러 로그 추출
2. LLM 수정 제안 저장
3. 적용된 코드 패치 저장
4. 에러 유형 자동 분류
5. Few-shot 데이터로 축적
```

### 2.4 MCP (Model Context Protocol) 연동

#### MCP 도구 목록
| 도구 | 설명 |
|------|------|
| `analyze_build_error` | 빌드 에러 분석 |
| `suggest_code_fix` | 코드 수정 제안 |
| `apply_patch` | 코드 패치 적용 |
| `validate_fix` | 수정 검증 |

#### MCP API
```yaml
POST /mcp/v1/tools/call
  Request:
    tool: "analyze_build_error"
    arguments:
      error_log: string
      source_files: array
  Response:
    result:
      error_type: string
      root_cause: string
      suggested_fixes: array
```

---

## 3. LLM 제공자

### 3.1 지원 LLM
| 제공자 | 모델 | 용도 | 우선순위 |
|--------|------|------|:--------:|
| OpenAI | GPT-4o | 주 분석/수정 | Primary |
| Anthropic | Claude 3.5 | 백업, 복잡한 분석 | Secondary |
| OpenAI | GPT-4o-mini | 단순 작업 | Fallback |

### 3.2 Fallback 전략
```
1차: GPT-4o (Primary)
     ↓ 실패/타임아웃
2차: Claude 3.5 (Secondary)
     ↓ 실패/타임아웃
3차: GPT-4o-mini (Fallback)
     ↓ 실패
4차: 수동 개입 요청
```

### 3.3 API 키 관리
- Azure Key Vault에 저장
- 환경별 분리 (dev/prd)
- 키 로테이션 지원

---

## 4. 비기능 요구사항

### 4.1 성능
| 항목 | 요구사항 |
|------|----------|
| LLM 응답 시간 | < 30초 |
| 최대 반복 수정 횟수 | 5회 |
| 동시 LLM 요청 | 10개 |

### 4.2 정확도
| 지표 | 목표 |
|------|------|
| 에러 분석 정확도 | > 95% |
| 수정 제안 성공률 | > 80% |
| Few-shot 적용 개선율 | > 10% |

### 4.3 비용 관리
| 항목 | 제한 |
|------|------|
| 프로젝트당 최대 토큰 | 100K tokens |
| 일일 API 호출 제한 | 1000회 |
| 월간 비용 예산 | 알림 설정 |

---

## 5. 보안 요구사항

| 항목 | 요구사항 |
|------|----------|
| 소스 코드 전송 | HTTPS 암호화 |
| 데이터 보관 | LLM 제공자에 저장 안 함 |
| API 키 보호 | Key Vault 사용 |
| 감사 로그 | 모든 LLM 호출 기록 |

---

## 6. 외부 의존성

| 의존성 | 설명 | 위험도 |
|--------|------|:------:|
| OpenAI API | 주 LLM 제공자 | 높음 |
| Claude API | 백업 LLM | 중간 |
| Azure Key Vault | API 키 관리 | 높음 |

---

## 변경 이력

| 버전 | 날짜 | 작성자 | 변경 내용 |
|------|------|--------|----------|
| 1.0.0 | 2026-01-13 | - | 기존 PRD에서 분리 |
