---
name: n8n-studio:summarize-workflow
description: Use this skill when the user wants to summarize an n8n workflow into a JSON file, analyze workflow node structure, or document what each node does. Triggers on requests like "n8n 워크플로우 요약해줘", "워크플로우 분석해서 JSON으로 저장해줘", "워크플로우 노드 요약 파일 만들어줘", or "summarize workflow [ID]".
---

# summarize-workflow

단일 책임: 지정된 n8n 워크플로우를 분석하여 노드 요약 JSON 파일로 저장한다.

## 수행 절차

### 1. 워크플로우 ID 확인

사용자가 워크플로우 ID를 제공하지 않은 경우:
- `mcp__n8n-mcp__n8n_list_workflows`로 전체 목록을 조회하여 사용자에게 선택 요청

### 2. 워크플로우 조회

`mcp__n8n-mcp__n8n_get_workflow`로 워크플로우 상세 정보를 가져온다.

### 3. 워크플로우 분석

조회된 워크플로우에서 다음 정보를 추출한다:

- **workflow_id**: 워크플로우의 고유 ID
- **workflow_name**: 워크플로우 이름
- **workflow_description**: 워크플로우 전체가 하는 일에 대한 1~3문장 요약 (없으면 노드 구조 기반으로 추론)
- **node_summary**: 각 노드에 대해:
  - **node_id**: 노드의 고유 ID (`id` 필드)
  - **node_name**: 노드 이름 (`name` 필드)
  - **node_summary**: 노드가 하는 일을 1~2문장으로 요약 (노드 타입, 파라미터, 연결 관계를 참고하여 추론)

### 4. 출력 경로 결정

현재 작업 디렉토리(Claude Code가 실행 중인 프로젝트 폴더)를 기준으로:

1. `workflow/` 폴더가 존재하는지 확인
2. **존재하는 경우**: `workflow/[프로젝트명]/summary/` 경로 사용
   - `[프로젝트명]`은 사용자에게 확인하거나 워크플로우 이름에서 유추
3. **존재하지 않는 경우**: 사용자에게 저장 위치를 직접 질문:
   ```
   workflow/ 폴더가 없습니다. JSON 파일을 어디에 저장할까요?
   예: docs/summaries/, summaries/, 또는 직접 경로 입력
   ```
   사용자가 지정한 경로에 `summary/` 하위 폴더를 생성하여 저장

### 5. JSON 파일 생성

파일명: `[workflow_id].json` (예: `JLvMyC2iPaAWePfH.json`)

저장 형식:

```json
{
  "workflow_id": "JLvMyC2iPaAWePfH",
  "workflow_name": "Task Board - Main Dispatcher",
  "workflow_description": "슬랙에서 수신된 slash command 또는 버튼 클릭 이벤트를 분류하여 적절한 서브 워크플로우로 라우팅한다.",
  "node_summary": [
    {
      "node_id": "webhook-001",
      "node_name": "Slack Webhook Trigger",
      "node_summary": "슬랙으로부터 POST 요청을 수신하는 진입점 노드. slash command 및 interactive payload를 모두 처리한다."
    },
    {
      "node_id": "router-001",
      "node_name": "Command Router",
      "node_summary": "수신된 이벤트 타입(slash command vs 버튼 클릭)을 분류하고 해당 서브 워크플로우로 실행을 위임한다."
    }
  ]
}
```

`summary/` 폴더가 없으면 자동으로 생성한다.

### 6. 완료 보고

저장된 파일 경로와 분석된 노드 수를 사용자에게 보고한다:

```
워크플로우 요약 완료:
- 워크플로우: [이름] (ID: [id])
- 분석된 노드 수: N개
- 저장 경로: workflow/[프로젝트명]/summary/[workflow_id].json
```
