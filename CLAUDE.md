# Claude Code 가이드라인

## 커밋 규칙

### 기본 원칙
- 논리적 단위별로 커밋
- [Conventional Commit Specification](https://www.conventionalcommits.org/) 준수
- 한국어로 작성

### 커밋 메시지 형식
```
<type>(<scope>): <subject>

<body>
```

### Type 종류
- `feat`: 새로운 기능 추가
- `fix`: 버그 수정
- `docs`: 문서 변경
- `style`: 코드 포맷팅, 세미콜론 누락 등 (코드 변경 없음)
- `refactor`: 리팩토링
- `test`: 테스트 추가/수정
- `chore`: 빌드, 설정 파일 등 기타 변경

### 주의사항
- footer에 불필요한 내용(작성 도구 정보 등) 포함하지 않음

## 워크로그 (Worklog)

워크로그는 `__worklog__` 디렉토리에 저장합니다.

### 파일명 형식
```
yyyyMMDD_hhmm - subject.md
```

- `yyyyMMDD`: 날짜 (예: 20260119)
- `hhmm`: 시간 (예: 1430)
- `subject`: 작업 주제

### 예시
```
20260119_1430 - API 엔드포인트 구현.md
20260119_1600 - 버그 수정.md
```
