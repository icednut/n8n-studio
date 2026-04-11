---
name: n8n-studio:develop
description: This skill should be used when the n8n-studio development cycle reaches the development phase (5단계), or when the user asks to "n8n 워크플로우 개발 진행해줘", "개발 단계 시작해줘". Reads the design document and builds or modifies n8n workflows using TDD for Code nodes.
argument-hint: "[docs 폴더 경로 (선택)]"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep", "Agent", "mcp__n8n-mcp__n8n_get_workflow", "mcp__n8n-mcp__n8n_create_workflow", "mcp__n8n-mcp__n8n_update_full_workflow", "mcp__n8n-mcp__n8n_update_partial_workflow", "mcp__n8n-mcp__n8n_validate_workflow", "mcp__n8n-mcp__search_nodes", "mcp__n8n-mcp__get_node"]
---

# n8n-studio:develop

5단계 개발을 담당한다. 설계 문서를 기반으로 n8n 워크플로우를 실제로 개발한다.

## 처리 흐름

### 설계 문서 확인

`docs/yyyyMMdd-기능요약/02-design.md`를 읽어 개발 계획을 파악한다.

### Code 노드 TDD 개발

Code 노드가 필요한 경우 workflow-developer 에이전트를 통해 `develop-code-node` 스킬로 TDD 방식으로 개발한다.

**TDD 순서 (방식 B — 로컬 먼저):**

1. `workflow/[프로젝트명]/test/unit/` 에 유닛 테스트 파일 작성 (`.test.js`)
2. `npm test`로 테스트 실행 → 실패 확인 (Red)
3. `workflow/[프로젝트명]/test/unit/` 에 구현 로직 파일 작성 (`.js`)
4. `npm test`로 테스트 실행 → 통과 확인 (Green)
5. 통과된 코드를 n8n Code 노드에 이식

**Code 노드 제약:**
- Execute Command 노드로 쉘 명령어 실행 금지 (특수한 경우 제외)
- Python 코드 노드 사용 금지
- 사용 가능한 외부 모듈: `simple-git`, `yaml`, `glob` (Dockerfile에 전역 설치됨)
- n8n 표현식 작성 시 `n8n-mcp-skills:n8n-expression-syntax` 참조

### 워크플로우 빌드

`build-workflow` 스킬로 n8n-mcp를 사용하여 워크플로우를 생성/수정한다.

**빌드 순서:**
1. 설계 문서의 노드 목록 확인
2. n8n-mcp 도구로 필요한 노드 타입 검색 및 설정 파악
3. 워크플로우 생성 또는 업데이트
4. 유효성 검사 실행

**n8n-mcp-skills 참조:**
- 노드 설정 시: `n8n-mcp-skills:n8n-node-configuration`
- JavaScript 작성 시: `n8n-mcp-skills:n8n-code-javascript`
- 표현식 작성 시: `n8n-mcp-skills:n8n-expression-syntax`
- 유효성 오류 해석 시: `n8n-mcp-skills:n8n-validation-expert`
- n8n-mcp 도구 사용 시: `n8n-mcp-skills:n8n-mcp-tools-expert`

### 자동 전환

개발 완료 후 리팩토링 필요 여부를 판단한다:
- 리팩토링이 필요하다고 판단되면 → `refactor` 스킬로 전환
- 불필요하다고 판단되면 → `verify` 스킬로 직접 전환
