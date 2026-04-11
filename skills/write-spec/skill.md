---
name: n8n-studio:write-spec
description: This skill should be used by n8n-studio agents when they need to write a specification document (spec.md) based on the analyzed workflow state and user requirements. Creates the docs/[project_name]/yyyyMMdd-feature/ directory and saves the spec.
---

# write-spec

단일 책임: 워크플로우 분석 결과와 사용자 요구사항을 바탕으로 명세 문서를 작성한다.

## 수행 작업

### docs 폴더 생성

```
docs/[프로젝트명]/yyyyMMdd-기능요약/
```

- 프로젝트명: 태스크 파일의 `[프로젝트 컨텍스트]`에서 전달받은 `project_name`
- 날짜: 작업 시작일 기준 `yyyyMMdd` 형식 (예: `20260410`)
- 기능 요약: 작업 내용을 2~4단어로 요약한 kebab-case 영문 (예: `slack-notification-fix`)
- 전체 예시: `docs/my-project/20260410-slack-notification-fix/`

### 01-spec.md 작성

`docs/[프로젝트명]/yyyyMMdd-기능요약/01-spec.md`를 아래 템플릿을 **그대로** 사용하여 작성한다. 섹션 이름, 순서, 형식을 임의로 변경하지 않는다.

```markdown
# 명세: [작업 제목]

## 메타 정보
- 요청 유형: [신규 개발 / 기능 변경 / 버그 수정]
- 대상 워크플로우: [이름 (ID)] (복수인 경우 줄 추가)
- Execution ID: [있는 경우 기재, 없으면 이 항목 삭제]
- 작업 일시: [yyyyMMdd]

## 요청 내용
[사용자 원문 기준으로 1~3문장. 의역하지 말고 핵심만 기술]

## 현재 상태
[analyze-workflow 결과 기반. 관련 노드/로직 현황을 bullet로 나열]
- [현재 상태 항목 1]
- [현재 상태 항목 2]

## 변경/추가 요구사항
1. [구체적인 요구사항 — 무엇을, 어떻게 변경/추가할지 명시]
2. [구체적인 요구사항]

## 제약 조건
- Execute Command 노드 사용 금지
- Python Code 노드 사용 금지
- [추가 제약이 있으면 기재, 없으면 위 두 줄만 유지]

## 완료 기준
- [ ] [테스트로 검증 가능한 기준 1]
- [ ] [테스트로 검증 가능한 기준 2]
```

**형식 규칙:**
- `Execution ID` 항목: 없으면 해당 줄을 아예 삭제한다 (빈 값으로 두지 않는다)
- `제약 조건`: 기본 2개(Execute Command, Python)는 항상 포함한다
- 자유 서술 금지: 각 섹션은 지정된 형식(bullet 또는 번호 목록)만 사용한다
- 불필요한 설명 문장 추가 금지

## 명세 완료 판단

명세 작성 후 아래를 확인한다:
- 대상 워크플로우가 명확히 특정됨
- 변경/추가할 내용이 구체적으로 기술됨
- 완료 기준이 검증 가능한 형태로 작성됨
- 제약 조건이 식별됨

이 조건이 모두 충족되면 명세 완료로 판단하고 git branch 생성으로 진행한다.
