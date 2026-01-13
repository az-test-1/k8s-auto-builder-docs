# Changelog: FTR-XXX [기능명]

이 문서는 [기능명] 기능의 모든 주요 변경사항을 기록합니다.

형식은 [Keep a Changelog](https://keepachangelog.com/ko/1.0.0/)를 따르며,
버전은 [Semantic Versioning](https://semver.org/lang/ko/)을 따릅니다.

---

## [Unreleased]

### Added
- [추가된 기능]

### Changed
- [변경된 기능]

### Fixed
- [수정된 버그]

---

## [1.1.0] - YYYY-MM-DD

### Added
- 새로운 검색 필터 옵션 추가 (#123)
- 벌크 작업 지원 (#124)

### Changed
- 기본 페이지 크기 20 → 50으로 변경 (#125)

### Deprecated
- `v1/old-endpoint` API 지원 중단 예정 (v2.0에서 제거)

### Fixed
- 특수문자 검색 시 오류 수정 (#126)

---

## [1.0.1] - YYYY-MM-DD

### Fixed
- 페이지네이션 오프셋 계산 오류 수정 (#120)
- 빈 결과 시 500 에러 반환 문제 해결 (#121)

### Security
- XSS 취약점 패치 (#122)

---

## [1.0.0] - YYYY-MM-DD

### Added
- 초기 기능 릴리즈
- 기본 CRUD 작업 지원
- 검색 및 필터링
- 페이지네이션

---

## 변경 유형 가이드

- **Added**: 새로운 기능
- **Changed**: 기존 기능 변경
- **Deprecated**: 곧 제거될 기능
- **Removed**: 제거된 기능
- **Fixed**: 버그 수정
- **Security**: 보안 취약점 수정
