---
name: n8n-studio:write-design
description: This skill should be used by n8n-studio agents when they need to write a design document (design.md) based on the spec. Documents workflow structure, node configuration, code snippets, and references relevant n8n-mcp-skills.
---

# write-design

단일 책임: 명세를 기반으로 n8n 워크플로우 설계 문서를 작성한다.

## 수행 작업

### 설계 원칙 적용

- **SRP 준수**: 워크플로우 1개 = 목표 작업 1개
- **금지 사항**: Execute Command 노드, Python 코드 노드
- **권장 사항**: JavaScript Code 노드, n8n 내장 노드 활용

### 02-design.md 작성

`docs/yyyyMMdd-기능요약/02-design.md` 내용:

```markdown
# 설계: [작업 제목]

## 변경 대상 워크플로우
| 워크플로우 | 변경 유형 | 설명 |
|-----------|---------|------|
| [이름]    | [신규/수정] | [설명] |

## 워크플로우 구조

### [워크플로우명]
**목적:** [이 워크플로우가 하는 단일 작업]

**노드 목록:**
| 노드 이름 | 노드 타입 | 역할 |
|---------|---------|------|
| ...     | ...     | ...  |

**흐름:**
[Trigger] → [Node1] → [Node2] → ... → [Output]

## Code 노드 설계 (해당 시)

### [노드 이름]
**역할:** [이 Code 노드가 하는 단일 작업]

**입력:** [입력 데이터 구조]
**출력:** [출력 데이터 구조]

**로직:**
```javascript
// 의사코드 또는 실제 코드 스니펫
```

## n8n-mcp-skills 참조 계획
- 노드 설정: `n8n-mcp-skills:n8n-node-configuration`
- [필요한 스킬 목록]

## 서브 워크플로우 연결 (해당 시)
[메인 워크플로우와 서브 워크플로우 간 연결 구조]
```

### 설계 시 참조할 스킬

필요에 따라 아래 스킬을 참조한다:
- 노드 설정 방법: `n8n-mcp-skills:n8n-node-configuration`
- 워크플로우 패턴: `n8n-mcp-skills:n8n-workflow-patterns`
- JavaScript 패턴: `n8n-mcp-skills:n8n-code-javascript`
- 표현식 문법: `n8n-mcp-skills:n8n-expression-syntax`

## 02-design.md 작성 금지 항목

아래 내용은 설계 문서에 포함하지 않는다:
- git 커밋 메시지 또는 커밋 계획
- PR 제목/본문
- 배포 절차 또는 릴리즈 노트
- 구현 완료 후 할 일 목록

이 정보는 `summarize-result` 스킬이 담당한다.
