---
name: n8n-studio:plan-tests
description: This skill should be used by n8n-studio agents when they need to plan and write integration test scenarios based on the design document. Creates test plan docs and actual test files in workflow/[프로젝트명]/test/unit/ and workflow/[프로젝트명]/test/acceptance/.
---

# plan-tests

단일 책임: 설계 문서를 바탕으로 테스트 시나리오를 계획하고 테스트 파일을 작성한다.

## 수행 작업

### 03-test-plan.md 작성

`docs/[프로젝트명]/yyyyMMdd-기능요약/03-test-plan.md` 내용:

```markdown
# 테스트 계획: [작업 제목]

## 단위 테스트 (Code 노드)
| 시나리오 ID | 대상 | 설명 | 파일 |
|-----------|------|------|------|
| UT-001    | [Code 노드명] | [테스트 목적] | workflow/[프로젝트명]/test/unit/xxx.test.js |

## 통합/인수 테스트
| 시나리오 ID | 워크플로우 | 설명 | 파일 |
|-----------|---------|------|------|
| IT-001    | [워크플로우명] | [테스트 목적] | workflow/[프로젝트명]/test/acceptance/xxx.json |
| AT-001    | [메인+서브] | [통합 테스트] | workflow/[프로젝트명]/test/acceptance/xxx.json |
```

### 단위 테스트 파일 작성

`workflow/[프로젝트명]/test/unit/` 에 `.test.js` 파일 작성:

```javascript
// workflow/[프로젝트명]/test/unit/[기능명].test.js
const { [함수명] } = require('./[기능명]');

describe('[Code 노드 이름]', () => {
  test('[시나리오 설명]', () => {
    const input = { /* n8n $input 형태 */ };
    const result = [함수명](input);
    expect(result).toEqual({ /* 기대 출력 */ });
  });
});
```

### 인수 테스트 파일 작성

`workflow/[프로젝트명]/test/acceptance/` 에 JSON 파일 작성:

```json
{
  "scenarioId": "IT-001",
  "workflowId": "[n8n 워크플로우 ID]",
  "description": "[시나리오 설명]",
  "input": {
    "triggerNode": "[트리거 노드 이름]",
    "data": { /* Test Execution 입력 JSON */ }
  },
  "expectedOutput": {
    "type": "json",
    "value": { /* 기대 출력 */ }
  }
}
```

파일 산출물 검증이 필요한 경우:
```json
{
  "expectedOutput": {
    "type": "file",
    "path": "[검증할 파일 경로]",
    "content": "[기대 파일 내용 또는 정규식 패턴]"
  }
}
```

## 테스트 커버리지 원칙

- 정상 케이스 (Happy Path): 반드시 포함
- 경계 케이스: 설계에서 식별된 제약 조건 기반
- 서브 워크플로우가 있는 경우: 메인+서브 통합 테스트 시나리오 포함
