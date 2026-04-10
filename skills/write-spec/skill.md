---
name: n8n-studio:write-spec
description: This skill should be used by n8n-studio agents when they need to write a specification document (spec.md) based on the analyzed workflow state and user requirements. Creates the docs/yyyyMMdd-feature/ directory and saves the spec.
---

# write-spec

단일 책임: 워크플로우 분석 결과와 사용자 요구사항을 바탕으로 명세 문서를 작성한다.

## 수행 작업

### docs 폴더 생성

```
docs/yyyyMMdd-기능요약/
```

- 날짜: 작업 시작일 기준 `yyyyMMdd` 형식 (예: `20260410`)
- 기능 요약: 작업 내용을 2~4단어로 요약한 kebab-case 영문 (예: `slack-notification-fix`)
- 전체 예시: `docs/20260410-slack-notification-fix/`

### spec.md 작성

`docs/yyyyMMdd-기능요약/spec.md` 내용:

```markdown
# 명세: [작업 제목]

## 메타 정보
- 요청 유형: [신규 개발 / 기능 변경 / 버그 수정]
- 대상 워크플로우: [이름 및 ID 목록]
- Execution ID: [있는 경우]
- 작업 일시: [yyyyMMdd]

## 요청 내용
[사용자가 요청한 내용을 구체적이고 간결하게 기술]

## 현재 상태
[analyze-workflow에서 파악한 현재 워크플로우 상태]

## 변경/추가 요구사항
1. [구체적인 요구사항 1]
2. [구체적인 요구사항 2]
...

## 제약 조건
- [기술적 제약, 비즈니스 규칙 등]

## 완료 기준
- [ ] [검증 가능한 완료 기준 1]
- [ ] [검증 가능한 완료 기준 2]
```

## 명세 완료 판단

명세 작성 후 아래를 확인한다:
- 대상 워크플로우가 명확히 특정됨
- 변경/추가할 내용이 구체적으로 기술됨
- 완료 기준이 검증 가능한 형태로 작성됨
- 제약 조건이 식별됨

이 조건이 모두 충족되면 명세 완료로 판단하고 git branch 생성으로 진행한다.
