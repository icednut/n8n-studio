---
name: n8n-studio:analyze-workflow
description: This skill should be used by n8n-studio agents when they need to analyze the current state of one or more n8n workflows, retrieve execution logs by Execution ID, or understand workflow structure and connections.
---

# analyze-workflow

단일 책임: n8n 워크플로우의 현재 상태를 분석하고 파악한다.

## 수행 작업

### 워크플로우 현황 파악

1. n8n-mcp의 `n8n_list_workflows`로 전체 워크플로우 목록 조회
2. 사용자가 지정한 워크플로우를 `n8n_get_workflow`로 상세 조회
3. 워크플로우 구조 파악:
   - 노드 목록 및 각 역할
   - 노드 간 연결 구조
   - 트리거 방식 (Webhook, Cron, Manual 등)
   - 서브 워크플로우 연결 여부

### Execution ID 기반 로그 분석 (버그 수정 시)

Execution ID가 있는 경우:
1. `n8n_executions`로 실행 로그 조회
2. 실패 지점 파악
3. 오류 메시지 및 스택 트레이스 분석
4. 문제 원인 가설 수립

### 연관 워크플로우 탐색

분석 중 다른 워크플로우와 연동되어 있다고 판단되면:
- Execute Workflow 노드 연결 추적
- Webhook 트리거 연결 추적
- 연관 워크플로우도 함께 분석

## 출력

분석 결과를 구조화된 형태로 정리하여 다음 단계(write-spec)에서 활용할 수 있도록 한다:
- 워크플로우 ID, 이름, 상태
- 주요 노드 목록과 역할
- 발견된 문제점 (버그 수정 시)
- 변경이 필요한 부분 식별
