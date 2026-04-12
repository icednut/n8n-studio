---
name: n8n-studio:analyze-workflow-input
description: Use when the user wants to analyze the trigger input data format of an n8n workflow from a past execution. Triggers on requests like "워크플로우 입력 양식 분석해줘", "Execution ID로 트리거 데이터 양식 파악해줘", "analyze-workflow-input [Execution ID]". Standalone skill - not part of the dev cycle.
argument-hint: "[Execution ID (선택)]"
allowed-tools: ["Read", "Write", "Edit", "Glob", "mcp__n8n-mcp__n8n_executions", "mcp__n8n-mcp__n8n_get_workflow"]
---

# analyze-workflow-input

단일 책임: 과거 실행(Execution) 데이터를 분석하여 워크플로우 첫 번째 노드의 입력 데이터 양식을 파악하고 summary 파일에 `triggerDataFormat`으로 저장한다.

## 주의 사항

이 스킬은 **개발 사이클 외부**에서 사용자가 직접 실행하는 standalone 스킬이다.
서브에이전트에서 실행하거나 개발 사이클에 통합하지 않는다.

## 수행 절차

### 1. Execution ID 확인

스킬 인수로 Execution ID가 전달되지 않은 경우, 사용자에게 요청한다:

```
분석할 실행(Execution)의 ID를 알려주세요.
n8n 대시보드 > Executions 에서 확인할 수 있습니다.
```

사용자가 Execution ID를 제공하지 않으면 즉시 작업을 중단하고 아래 메시지를 출력한다:

```
Execution ID가 없으면 트리거 데이터 양식을 파악할 수 없습니다. 작업을 종료합니다.
```

### 2. Execution 데이터 조회

`mcp__n8n-mcp__n8n_executions`를 사용하여 해당 Execution의 상세 데이터를 가져온다.

- 실행된 워크플로우 ID 확인
- 각 노드의 실행 결과 데이터 수집

### 3. 워크플로우 구조 파악

`mcp__n8n-mcp__n8n_get_workflow`로 워크플로우를 조회하여 첫 번째 노드(트리거 노드)를 파악한다.

첫 번째 노드 판별 기준:
- 다른 노드로부터 연결을 받지 않는 노드
- 노드 타입이 트리거 계열인 노드 (`n8n-nodes-base.webhook`, `n8n-nodes-base.manualTrigger`, `n8n-nodes-base.scheduleTrigger`, `n8n-nodes-base.executeWorkflowTrigger` 등)

### 4. 트리거 타입별 처리

**Webhook 노드인 경우:**

해당 Execution에서 Webhook 노드의 **Output 데이터**를 추출한다.
`body`, `headers`, `params`, `query` 구조를 그대로 샘플 데이터로 활용한다.

예시:
```json
{
  "body": {
    "user_id": "U123456",
    "command": "/task",
    "text": "add buy milk",
    "payload": {
      "type": "block_actions",
      "actions": [
        { "action_id": "btn_complete", "value": "task_001" }
      ]
    }
  },
  "headers": {
    "content-type": "application/x-www-form-urlencoded"
  },
  "params": {}
}
```

중첩 JSON(nested JSON)은 별도 표기 없이 실제 구조 그대로 저장한다.

**Webhook 노드가 아닌 경우 (Manual Trigger, Schedule Trigger 등):**

자동 추출이 불가하므로 사용자에게 데이터 양식을 직접 요청한다:

```
이 워크플로우의 첫 번째 노드는 [노드명] ([노드타입]) 입니다.
Webhook 노드가 아니므로 실행 데이터에서 자동 추출이 불가합니다.

테스트 시 사용할 트리거 입력 데이터를 JSON 형태로 알려주세요.
실제 운영에서 사용하는 데이터 예시를 기반으로 작성해 주시면 됩니다.

예:
{
  "param1": "실제값1",
  "param2": {
    "nested_key": "실제값2"
  }
}
```

사용자가 JSON을 제공하면 그것을 `triggerDataFormat`으로 사용한다.
사용자가 제공하지 않으면 아래 메시지를 출력하고 작업을 중단한다:

```
트리거 데이터 양식을 제공받지 못했습니다. 작업을 종료합니다.
triggerDataFormat 없이는 통합 테스트 시 테스트 데이터를 자동 생성할 수 없습니다.
```

### 5. Summary 파일 업데이트

`workflow/*/summary/[workflow_id].json` 파일을 찾아 `triggerDataFormat` 필드를 추가(또는 업데이트)한다.

Summary 파일이 없는 경우 아래 메시지를 출력하고 중단한다:

```
workflow/.../summary/[workflow_id].json 파일이 없습니다.
먼저 n8n-studio:summarize-workflow 스킬을 실행하여 summary 파일을 생성해 주세요.
```

저장 후 전체 파일 형식:

```json
{
  "workflow_id": "JLvMyC2iPaAWePfH",
  "workflow_name": "Task Board - Main Dispatcher",
  "workflow_description": "슬랙에서 수신된 slash command를 분류하여 서브 워크플로우로 라우팅한다.",
  "triggerDataFormat": {
    "body": {
      "user_id": "U123456",
      "command": "/task",
      "text": "add buy milk",
      "team_id": "T78901",
      "payload": {
        "type": "block_actions",
        "actions": [
          { "action_id": "btn_complete", "value": "task_001" }
        ]
      }
    },
    "headers": {
      "content-type": "application/x-www-form-urlencoded"
    },
    "params": {}
  },
  "node_summary": [
    {
      "node_id": "webhook-001",
      "node_name": "Slack Webhook",
      "node_summary": "슬랙으로부터 POST 요청을 수신하는 진입점 노드."
    }
  ]
}
```

### 6. 완료 보고

```
triggerDataFormat 저장 완료:
- 워크플로우: [이름] (ID: [id])
- 트리거 노드: [노드명] ([노드타입])
- 저장 경로: workflow/[프로젝트명]/summary/[workflow_id].json

이제 plan-tests 실행 시 이 샘플 데이터를 기반으로 시나리오별 테스트 데이터가 자동 생성됩니다.
```
