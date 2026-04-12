---
name: n8n-studio:build-workflow
description: This skill should be used by n8n-studio agents when they need to create or modify n8n workflows using n8n-mcp tools based on the design document. Handles workflow creation, node configuration, and validation.
---

# build-workflow

단일 책임: 설계 문서를 기반으로 n8n-mcp를 사용하여 워크플로우를 생성하거나 수정한다.

## 수행 작업

### triggerDataFormat 확인 (빌드 전 선행)

워크플로우 생성/수정 전에 해당 워크플로우의 summary 파일에서 `triggerDataFormat`을 확인한다:

1. `workflow/*/summary/[workflow_id].json` 파일을 `Glob`으로 탐색
2. `triggerDataFormat` 필드가 있으면 첫 번째 노드(Webhook, Manual Trigger 등) 설정 시 참고 자료로 활용한다:
   - Webhook 노드: `triggerDataFormat.body`의 필드 구조를 기반으로 이후 노드의 `$json.body.*` 표현식을 작성한다
   - 트리거 노드 Output의 실제 데이터 형태를 파악하여 다음 노드에서의 데이터 접근 방식을 결정한다
3. `triggerDataFormat`이 없으면 설계 문서의 입력 명세만으로 진행한다

### 노드 타입 탐색

설계 문서의 노드 목록을 확인하고 필요한 노드 타입을 탐색한다:
- `n8n-mcp-skills:n8n-mcp-tools-expert` 참조하여 적절한 검색 방법 선택
- 노드 설정 세부 사항: `n8n-mcp-skills:n8n-node-configuration` 참조

### 신규 워크플로우 생성

`n8n_create_workflow` 또는 `n8n_generate_workflow` 사용:
- 워크플로우 이름은 목적을 명확히 표현하는 이름 사용
- 노드 연결 구조는 설계 문서의 흐름도 기반

### 기존 워크플로우 수정

`n8n_update_partial_workflow` (부분 수정) 또는 `n8n_update_full_workflow` (전체 교체):
- 부분 수정: 특정 노드 추가/변경/삭제 시
- 전체 교체: 구조가 크게 변경될 시

### 유효성 검사

수정 후 반드시 `n8n_validate_workflow` 실행:
- 유효성 오류 발생 시 `n8n-mcp-skills:n8n-validation-expert` 참조하여 해결

### 검증 확인

워크플로우 빌드 완료 후 n8n 서버에서 실제 구조 재확인:
- `n8n_get_workflow`로 저장된 결과 확인
- 설계 문서의 노드 목록과 대조

## n8n-mcp-skills 참조 가이드

| 상황 | 참조할 스킬 |
|------|-----------|
| 어떤 n8n-mcp 도구를 써야 할지 모를 때 | `n8n-mcp-tools-expert` |
| 특정 노드 설정 파라미터를 모를 때 | `n8n-node-configuration` |
| 워크플로우 구조 패턴이 필요할 때 | `n8n-workflow-patterns` |
| 유효성 검사 오류가 이해 안될 때 | `n8n-validation-expert` |
| JavaScript 코드 작성 시 | `n8n-code-javascript` |
| n8n 표현식 작성 시 | `n8n-expression-syntax` |
