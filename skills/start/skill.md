---
name: n8n-studio:start
description: This skill should be used when the user asks to "n8n 워크플로우 개발해줘", "n8n 워크플로우 만들어줘", "n8n 워크플로우 기능 추가해줘", "n8n 워크플로우 수정해줘", "n8n 워크플로우 버그 수정해줘", "n8n 워크플로우 고쳐줘", or any request related to creating, modifying, or fixing n8n workflows. This is the main entry point and full orchestrator for the n8n-studio development cycle.
argument-hint: "[개발/수정/버그수정 요청 설명]"
allowed-tools: ["Read", "Write", "Bash", "Glob", "Grep", "Agent", "mcp__n8n-mcp__n8n_list_workflows", "mcp__n8n-mcp__n8n_get_workflow", "mcp__n8n-mcp__n8n_executions"]
---

# n8n-studio:start

전체 8단계 개발 사이클의 오케스트레이터. 각 단계를 **독립 서브에이전트로 디스패치**하여 컨텍스트를 격리한다.

## 오케스트레이션 원칙

**오케스트레이터(이 스킬)는 최소 상태만 유지한다:**
- 각 단계는 독립 Agent로 실행 → 이전 단계 컨텍스트를 상속받지 않음
- 에이전트 간 정보 전달은 **파일을 통해서만** 이루어짐
- 에이전트는 완료 후 **간결한 상태만 반환** (파일 경로, 성공/실패)
- 오케스트레이터는 아래 4가지만 추적:

```
docs_path:    docs/yyyyMMdd-기능요약/
branch:       feature/작업요약
workflow_ids: [id1, id2, ...]
ralf_cycle:   1
```

---

## 1단계: 요청 유형 분류 (직접 처리)

사용자 요청을 분석하여 분류한다:
- **신규 개발**: 새 워크플로우 생성
- **기능 변경/추가**: 기존 워크플로우 수정
- **버그 수정**: 오류 또는 의도치 않은 동작 수정

## 2단계: 정보 수집 (직접 처리 — 사용자 인터랙션)

사용자에게 질문한다:
- 관련 n8n 워크플로우 이름 또는 ID (복수 가능)
- 버그 수정 요청인 경우: Execution ID 유무

n8n-mcp로 지정된 워크플로우를 조회하여 ID를 확인한다. 연관 워크플로우가 있다고 판단되면 함께 조회한다.

---

## 3단계: workflow-planner 에이전트 디스패치

아래 형식으로 Agent를 호출한다. **프롬프트에 필요한 정보만 포함한다:**

```
Agent(prompt="""
당신은 n8n 워크플로우 플래닝 전문가입니다.
workflow-planner 에이전트로서 아래 정보를 바탕으로 명세를 작성하세요.

[입력 정보]
- 요청 유형: [신규 개발 / 기능 변경 / 버그 수정]
- 대상 워크플로우: [이름: ID 목록]
- Execution ID: [있으면 기재, 없으면 없음]
- 사용자 요청: [요약]
- 진입 유형: [초기 진입 / 재진입 (실패 시나리오: ...)]

[작업]
1. analyze-workflow 스킬로 워크플로우를 분석하세요
2. [초기 진입] 요구사항이 불명확하면 사용자와 대화하여 구체화하세요
   [재진입] 사용자 대화 없이 이슈 기반으로 바로 진행하세요
3. write-spec 스킬로 docs/yyyyMMdd-기능요약/spec.md를 작성하세요
4. git checkout -b feature/작업요약 으로 브랜치를 생성하세요

[반환]
완료 시 아래 형식으로만 반환하세요:
docs_path: docs/[생성한 폴더명]/
branch: feature/[브랜치명]
""")
```

반환받은 `docs_path`와 `branch`를 오케스트레이터 상태에 저장한다.

---

## 4단계: workflow-designer 에이전트 디스패치

```
Agent(prompt="""
당신은 n8n 워크플로우 설계 전문가입니다.
workflow-designer 에이전트로서 아래 파일을 읽고 설계를 수행하세요.

[입력 파일]
- 명세 문서: [docs_path]/spec.md

[작업]
1. spec.md를 읽으세요
2. write-design 스킬로 [docs_path]/design.md를 작성하세요
3. plan-tests 스킬로 테스트 파일을 작성하세요:
   - workflow/test/unit/ (단위 테스트)
   - workflow/test/acceptance/ (인수 테스트 JSON)
   - [docs_path]/test-plan.md (테스트 계획)

[반환]
완료 시 아래 형식으로만 반환하세요:
design_done: true
test_files: [파일 목록, 쉼표 구분]
""")
```

---

## 5단계: workflow-developer 에이전트 디스패치

```
Agent(prompt="""
당신은 n8n 워크플로우 개발 전문가입니다.
workflow-developer 에이전트로서 아래 파일을 읽고 개발을 수행하세요.

[입력 파일]
- 설계 문서: [docs_path]/design.md

[작업]
1. design.md를 읽으세요
2. Code 노드가 필요하면 develop-code-node 스킬로 TDD 개발하세요
   (workflow/test/unit/ 에 테스트 작성 → npm test → 통과 → n8n 이식)
3. build-workflow 스킬로 n8n 워크플로우를 생성/수정하세요
4. 리팩토링이 필요하다고 판단되면 수행하세요 (불필요하면 건너뜀)

[반환]
완료 시 아래 형식으로만 반환하세요:
dev_done: true
workflow_ids: [개발/수정된 워크플로우 ID 목록]
refactored: [true/false]
""")
```

반환받은 `workflow_ids`를 오케스트레이터 상태에 저장한다.

---

## 7단계: workflow-verifier 에이전트 디스패치 (RALF 루프)

`ralf_cycle` 카운터를 초기화(1)하고 아래 루프를 수행한다.

```
Agent(prompt="""
당신은 n8n 워크플로우 검증 전문가입니다.
workflow-verifier 에이전트로서 통합 테스트를 실행하세요.

[입력]
- 테스트 계획: [docs_path]/test-plan.md
- 테스트 파일 경로: workflow/test/acceptance/
- 대상 워크플로우 ID: [workflow_ids]
- 현재 사이클: [ralf_cycle]/3

[작업]
1. run-verification 스킬로 모든 시나리오를 처음부터 실행하세요
2. 결과를 판단하세요

[반환]
완료 시 아래 형식으로만 반환하세요:
all_passed: [true/false]
failed_scenarios: [실패 시나리오 ID 목록, 없으면 빈 배열]
failure_summary: [실패 원인 요약, 통과 시 빈 문자열]
""")
```

**결과 처리:**

- `all_passed: true` → 8단계로 진행
- `all_passed: false` AND `ralf_cycle < 3`:
  1. 실패 목록을 사용자에게 표시
  2. "재시도합니다 (사이클 [N]/3)" 알림
  3. `ralf_cycle += 1`
  4. 3단계로 돌아가 workflow-planner를 재진입 모드로 디스패치 (실패 시나리오 전달)
  5. 4단계 → 5단계 → 7단계 반복
- `all_passed: false` AND `ralf_cycle == 3`:
  아래 형식으로 보고하고 사용자 지침 대기:
  ```
  ## 검증 실패 보고 (사이클 3/3)
  ### 실패한 시나리오
  [failure_summary 내용]
  사용자 지침이 필요합니다. 어떻게 진행할까요?
  ```
  지침 수신 후 `ralf_cycle = 1`로 초기화하여 3단계부터 재시작 (횟수 제한 없음)

---

## 8단계: workflow-finisher 에이전트 디스패치

```
Agent(prompt="""
당신은 n8n 워크플로우 작업 마무리 전문가입니다.
workflow-finisher 에이전트로서 작업을 마무리하세요.

[입력]
- 문서 폴더: [docs_path]
- 완료된 워크플로우 ID: [workflow_ids]
- git 브랜치: [branch]
- RALF 사이클 횟수: [ralf_cycle]

[작업]
summarize-result 스킬로 아래를 수행하세요:
1. [docs_path]/result.md 작성
2. 워크플로우 JSON을 workflow/ 폴더에 저장
3. git add/commit/push
4. GitHub PR 생성 (base: main)

[반환]
완료 시 PR URL만 반환하세요.
""")
```

PR URL을 사용자에게 보고하며 완료한다.

---

## 중단 조건

아래 상황에서만 사용자 입력을 기다린다:
- 2단계: 워크플로우 정보 질문
- 3단계(초기): 요구사항이 불명확하여 사용자 대화가 필요한 경우
- 7단계: 3회 사이클 실패 후 지침 대기
- 그 외 모든 상황은 멈추지 않고 자동으로 진행한다
