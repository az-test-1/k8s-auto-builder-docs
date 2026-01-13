# Screen Specifications (화면 정의서)

> Auto-Builder UI 화면별 상세 스펙 문서

## 개요

이 폴더에는 Auto-Builder 시스템의 모든 UI 화면에 대한 상세 정의서가 포함되어 있습니다. 각 문서는 화면 구성, 컴포넌트, API 연동, 상태 관리 등을 정의합니다.

## 문서 목록

### 공통 화면 (SCR-00x)
| 문서 | 화면명 | URL | Parent PRD |
|------|--------|-----|------------|
| [SCR-001](./SCR-001-Login.md) | 로그인 | `/login` | PRD-030 |
| [SCR-002](./SCR-002-Layout.md) | 레이아웃 | - | PRD-030 |

### 대시보드 (SCR-01x)
| 문서 | 화면명 | URL | Parent PRD |
|------|--------|-----|------------|
| [SCR-010](./SCR-010-Dashboard.md) | 대시보드 | `/dashboard` | PRD-030 |

### 프로젝트 관리 (SCR-02x)
| 문서 | 화면명 | URL | Parent PRD |
|------|--------|-----|------------|
| [SCR-020](./SCR-020-ProjectList.md) | 프로젝트 목록 | `/projects` | PRD-010 |
| [SCR-021](./SCR-021-ProjectCreate.md) | 프로젝트 생성 | `/projects/new` | PRD-010 |
| [SCR-022](./SCR-022-ProjectDetail.md) | 프로젝트 상세 | `/projects/:id` | PRD-010 |

### 실행 관리 (SCR-03x)
| 문서 | 화면명 | URL | Parent PRD |
|------|--------|-----|------------|
| [SCR-030](./SCR-030-ExecutionRun.md) | 실행 | `/projects/:id/run` | PRD-010 |
| [SCR-031](./SCR-031-ExecutionMonitor.md) | 실행 모니터링 | `/executions/:id` | PRD-010 |
| [SCR-032](./SCR-032-ExecutionHistory.md) | 실행 이력 | `/projects/:id/history` | PRD-010 |
| [SCR-033](./SCR-033-ExecutionResult.md) | 실행 결과 | `/executions/:id/result` | PRD-010 |

### 관리자 (SCR-04x)
| 문서 | 화면명 | URL | Parent PRD |
|------|--------|-----|------------|
| [SCR-040](./SCR-040-UserManagement.md) | 사용자 관리 | `/admin/users` | PRD-030 |
| [SCR-041](./SCR-041-GroupManagement.md) | 그룹 관리 | `/admin/groups` | PRD-030 |
| [SCR-042](./SCR-042-SystemSettings.md) | 시스템 설정 | `/admin/settings` | PRD-030 |

### AI/LLM 관리 (SCR-05x)
| 문서 | 화면명 | URL | Parent PRD |
|------|--------|-----|------------|
| [SCR-050](./SCR-050-PromptManagement.md) | 프롬프트 관리 | `/admin/prompts` | PRD-020 |
| [SCR-051](./SCR-051-FewshotManagement.md) | Few-shot 관리 | `/admin/fewshots` | PRD-020 |

## 문서 구조

각 화면 정의서는 다음 구조를 따릅니다:

```
# SCR-XXX: 화면명
1. 개요
   - 화면 정보 (ID, URL, 접근 권한)
   - 목적
2. 화면 구성
   - 와이어프레임 (ASCII)
   - 컴포넌트 목록
3. 컴포넌트 상세
   - 각 컴포넌트별 Props, Events
4. API 연동
   - 엔드포인트, 요청/응답 스펙
5. 상태 관리
   - State 구조, Actions
6. 에러 처리
7. 접근성 (a11y)
8. 반응형 디자인
```

## 번호 체계

| 범위 | 카테고리 | 설명 |
|------|----------|------|
| 001-009 | Common | 공통 화면 (로그인, 레이아웃) |
| 010-019 | Dashboard | 대시보드, 통계 |
| 020-029 | Project | 프로젝트 CRUD |
| 030-039 | Execution | 실행, 모니터링, 결과 |
| 040-049 | Admin | 사용자/그룹/시스템 관리 |
| 050-059 | AI/LLM | 프롬프트, Few-shot 관리 |

## 관련 문서

- [PRD-010-Core-Migration](../01-prd/PRD-010-Core-Migration.md) - 프로젝트/실행 기능
- [PRD-020-AI-LLM](../01-prd/PRD-020-AI-LLM.md) - AI/LLM 기능
- [PRD-030-Admin-Dashboard](../01-prd/PRD-030-Admin-Dashboard.md) - 관리자/대시보드 기능
