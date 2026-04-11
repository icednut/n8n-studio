---
name: n8n-studio:analyze-workflow
description: This skill should be used by n8n-studio agents when they need to analyze the current state of one or more n8n workflows, retrieve execution logs by Execution ID, or understand workflow structure and connections.
---

# analyze-workflow

단일 책임: n8n 워크플로우의 현재 상태를 분석하고 파악한다.

## 수행 작업

### 워크플로우 현황 파악 (summary 파일 기반)

1. 현재 프로젝트 폴더에서 `workflow/[프로젝트명]/summary/` 경로를 탐색
   - `Glob`으로 `workflow/*/summary/*.json` 패턴 검색
   - 발견된 JSON 파일들을 모두 읽어 workflow_id, workflow_name, node_summary 목록 확인

2. 사용자가 지정한 워크플로우를 summary 파일에서 매칭:
   - workflow_id 또는 workflow_name 기준으로 탐색
   - 매칭된 summary에서 노드 구조 추출 (node_id, node_name, node_summary)

3. summary 파일이 없는 경우:
   - 사용자에게 `n8n-studio:summarize-workflow` 스킬 실행을 요청
   - 또는 `n8n_list_workflows`로 워크플로우 목록만 조회하여 ID 확인 후 사용자에게 안내

### Execution ID 기반 로그 분석 (버그 수정 시)

Execution ID가 있는 경우:
1. `n8n_executions`로 실행 로그 조회
2. 실패 지점 파악
3. 오류 메시지 및 스택 트레이스 분석
4. 문제 원인 가설 수립

### 신규 개발 시 유사 워크플로우 탐색

요청 유형이 신규 개발인 경우:
1. `workflow/*/summary/` 하위의 모든 summary JSON 파일을 스캔
2. workflow_name, workflow_description, node_summary를 키워드 기반으로 비교
3. 신규 개발 목적과 유사한 워크플로우를 최대 3개 선별하여 참고 목록 제공:
   ```
   참고 가능한 유사 워크플로우:
   - [workflow_name] (ID: [workflow_id]): [유사한 이유]
   - ...
   ```

### 연관 워크플로우 탐색

summary 파일 분석 중 다른 워크플로우와 연동되어 있다고 판단되면:
- 해당 워크플로우의 summary 파일도 함께 읽어 구조 파악

## 출력

분석 결과를 구조화된 형태로 정리하여 다음 단계(write-spec)에서 활용할 수 있도록 한다:
- 워크플로우 ID, 이름, 상태
- 주요 노드 목록과 역할 (summary 파일 기반)
- 발견된 문제점 (버그 수정 시)
- 변경이 필요한 부분 식별
- 신규 개발 시 참고 가능한 유사 워크플로우 목록
