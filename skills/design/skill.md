---
name: n8n-studio:design
description: This skill should be used when the n8n-studio development cycle reaches the design phase (4단계), or when the user asks to "n8n 워크플로우 설계해줘", "설계 단계 진행해줘". Reads the spec document and produces design documents and test scenarios.
argument-hint: "[docs 폴더 경로 (선택)]"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep", "Agent", "mcp__n8n-mcp__n8n_get_workflow", "mcp__n8n-mcp__search_nodes", "mcp__n8n-mcp__get_node"]
---

# n8n-studio:design

4단계 설계를 담당한다. `docs/yyyyMMdd-기능요약/spec.md`를 읽고 설계 문서와 통합 테스트 시나리오를 산출한다.

## 처리 흐름

### 설계 문서 작성

workflow-designer 에이전트를 호출하여 `write-design` 스킬로 설계 문서를 작성한다.

**설계 원칙:**
- n8n 워크플로우 1개 = 하나의 목표 작업만 (Single Responsibility Principle)
- Execute Command 노드 사용 금지 (특수한 경우 제외)
- Python Code 노드 사용 금지
- JavaScript Code 노드 사용 시 복잡한 로직은 코드 스니펫으로 설계 문서에 명시

**설계 문서 내용:**
- 워크플로우 변경/추가 항목 목록
- 각 노드의 역할과 설정 방향
- 노드 간 연결 구조
- Code 노드가 필요한 경우 로직 의사코드 또는 코드 스니펫
- 사용할 n8n 노드 타입 목록

설계 문서 저장 경로: `docs/yyyyMMdd-기능요약/design.md`

### 통합 테스트 시나리오 작성

`plan-tests` 스킬로 테스트 시나리오를 작성한다.

**테스트 시나리오 포함 내용:**
- 시나리오 이름과 목적
- n8n Test Execution 입력 파라미터 JSON
- 기대 출력값 (필요 시)
- 파일 산출물이 있는 경우: 기대 파일 내용 또는 검증 방법
- 단위 테스트 시나리오 (`workflow/test/unit/`)
- 통합/인수 테스트 시나리오 (`workflow/test/acceptance/`)

테스트 시나리오 파일: `docs/yyyyMMdd-기능요약/test-plan.md` (계획 문서)
실제 테스트 코드: `workflow/test/unit/`, `workflow/test/acceptance/`

### 자동 전환

설계 문서와 테스트 시나리오 작성 완료 후 자동으로 `develop` 스킬로 전환한다.

## n8n-mcp-skills 참조

- 노드 설정 설계 시: `n8n-mcp-skills:n8n-node-configuration`
- 워크플로우 구조 설계 시: `n8n-mcp-skills:n8n-workflow-patterns`
- JavaScript 코드 스니펫 설계 시: `n8n-mcp-skills:n8n-code-javascript`
- 표현식 설계 시: `n8n-mcp-skills:n8n-expression-syntax`
