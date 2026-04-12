---
name: n8n-studio:plan-tests
description: This skill should be used by n8n-studio agents when they need to plan and write integration test scenarios based on the design document. Creates test plan docs and actual test files in workflow/[프로젝트명]/test/unit/ and workflow/[프로젝트명]/test/integration/.
---

# plan-tests

단일 책임: 설계 문서를 바탕으로 테스트 시나리오를 계획하고 테스트 파일을 작성한다.

## 테스트 범위

- **단위 테스트 (Code 노드)**: 기존과 동일하게 유지
- **통합 테스트**: 워크플로우 단위로 Test Execution으로 검증 (인수 테스트는 진행하지 않음)

## 수행 작업

### 03-test-plan.md 작성

`docs/[프로젝트명]/yyyyMMdd-기능요약/03-test-plan.md` 내용:

```markdown
# 테스트 계획: [작업 제목]

## 단위 테스트 (Code 노드)
| 시나리오 ID | 대상 | 설명 | 파일 |
|-----------|------|------|------|
| UT-001    | [Code 노드명] | [테스트 목적] | workflow/[프로젝트명]/test/unit/xxx.test.js |

## 통합 테스트
| 시나리오 ID | 워크플로우 | 설명 | 파일 |
|-----------|---------|------|------|
| IT-001    | [워크플로우명] | [테스트 목적] | workflow/[프로젝트명]/test/integration/xxx.json |
| IT-002    | [서브 워크플로우명] | [독립 테스트] | workflow/[프로젝트명]/test/integration/xxx.json |
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

### 통합 테스트 파일 작성

`workflow/[프로젝트명]/test/integration/` 에 JSON 파일 작성.

**원칙:**
- 한 번에 하나의 워크플로우만 Test Execution으로 검증한다
- 워크플로우 A가 워크플로우 B를 호출하는 경우, A와 B 각각 별도 통합 테스트를 작성한다
- A 테스트 시 B 호출 여부(서브 워크플로우 호출 검증)만 확인하고 B의 실제 동작은 검증하지 않는다

**테스트 데이터 준비 (triggerDataFormat 활용):**

통합 테스트 파일 작성 전에 해당 워크플로우의 `workflow/[프로젝트명]/summary/[workflow_id].json`을 읽어 `triggerDataFormat` 여부를 확인한다.

- `triggerDataFormat`이 **있는 경우**: 해당 샘플 데이터를 기반으로 시나리오별 테스트 데이터를 생성한다. 샘플 데이터의 필드 구조와 타입을 유지하면서 시나리오 목적에 맞게 값을 변형하여 여러 테스트 케이스를 만든다.
- `triggerDataFormat`이 **없는 경우**: 설계 문서의 입력 명세를 바탕으로 테스트 데이터를 직접 작성한다.

**시나리오별 데이터 변형 예시 (triggerDataFormat 기반):**

샘플 데이터:
```json
{
  "body": { "command": "/task", "text": "add buy milk", "user_id": "U123456" }
}
```

→ IT-001 (정상 케이스): `text: "add buy milk"` (샘플 그대로)
→ IT-002 (경계 케이스): `text: ""` (빈 텍스트)
→ IT-003 (다른 커맨드): `text: "list"` (다른 명령)

**파일 형식 (Webhook 트리거 워크플로우):**

```json
{
  "scenarioId": "IT-001",
  "workflowId": "[n8n 워크플로우 ID]",
  "workflowName": "[워크플로우 이름]",
  "description": "[시나리오 설명]",
  "triggerType": "webhook",
  "input": {
    "triggerNode": "[Webhook 노드 이름]",
    "data": {
      "body": { /* triggerDataFormat.body 기반으로 시나리오에 맞게 생성 */ },
      "headers": { /* triggerDataFormat.headers 기반 */ },
      "params": {}
    }
  },
  "expectedOutput": {
    "type": "json",
    "value": { /* 기대 출력 */ }
  }
}
```

**파일 형식 (서브 워크플로우 호출 검증 포함):**

워크플로우 A가 B를 호출하는 경우, A의 통합 테스트에 `subWorkflowVerification`을 추가한다.
B의 실제 실행 결과는 검증하지 않으며, A의 "Execute Workflow" 노드가 올바른 파라미터로 호출되었는지만 확인한다.

```json
{
  "scenarioId": "IT-001",
  "workflowId": "[워크플로우 A의 ID]",
  "workflowName": "[워크플로우 A 이름]",
  "description": "[시나리오 설명]",
  "triggerType": "webhook",
  "input": {
    "triggerNode": "[Webhook 노드 이름]",
    "data": { /* 테스트 데이터 */ }
  },
  "expectedOutput": {
    "type": "json",
    "value": { /* A의 최종 기대 출력 */ }
  },
  "subWorkflowVerification": [
    {
      "nodeName": "[Execute Workflow 노드 이름]",
      "calledWorkflowId": "[워크플로우 B의 ID]",
      "expectedInputData": { /* B에 전달될 기대 입력 파라미터 */ }
    }
  ]
}
```

**파일 형식 (Webhook 외 트리거):**

```json
{
  "scenarioId": "IT-001",
  "workflowId": "[n8n 워크플로우 ID]",
  "workflowName": "[워크플로우 이름]",
  "description": "[시나리오 설명]",
  "triggerType": "manual",
  "input": {
    "triggerNode": "[트리거 노드 이름]",
    "data": { /* triggerDataFormat 또는 설계 명세 기반 */ }
  },
  "expectedOutput": {
    "type": "json",
    "value": { /* 기대 출력 */ }
  }
}
```

## 테스트 커버리지 원칙

- 정상 케이스 (Happy Path): 반드시 포함. `triggerDataFormat` 샘플 데이터를 기본 케이스로 사용
- 경계 케이스: 설계에서 식별된 제약 조건 기반. 샘플 데이터의 특정 필드 값을 변형하여 생성
- 서브 워크플로우 호출이 있는 경우: 해당 워크플로우는 별도 통합 테스트 파일 작성
