---
name: n8n-studio:start
description: This skill should be used when the user asks to "n8n 워크플로우 개발해줘", "n8n 워크플로우 만들어줘", "n8n 워크플로우 기능 추가해줘", "n8n 워크플로우 수정해줘", "n8n 워크플로우 버그 수정해줘", "n8n 워크플로우 고쳐줘", or any request related to creating, modifying, or fixing n8n workflows. This is the main entry point and full orchestrator for the n8n-studio development cycle.
argument-hint: "[개발/수정/버그수정 요청 설명]"
allowed-tools: ["Read", "Write", "Bash", "Glob", "Grep", "mcp__n8n-mcp__n8n_list_workflows", "mcp__n8n-mcp__n8n_get_workflow", "mcp__n8n-mcp__n8n_executions"]
---

# n8n-studio:start

전체 8단계 개발 사이클의 오케스트레이터. 각 단계를 **독립 Tmux 창에서 실행되는 인터랙티브 Claude 세션으로 디스패치**하여 컨텍스트를 격리하고 진행상황을 실시간으로 확인할 수 있게 한다.

## 오케스트레이션 원칙

**오케스트레이터(이 스킬)는 최소 상태만 유지한다:**
- 각 단계는 새 Tmux 창의 인터랙티브 Claude 세션으로 실행 → 이전 단계 컨텍스트를 상속받지 않음
- 에이전트 간 정보 전달은 **파일을 통해서만** 이루어짐
- 오케스트레이터는 아래 5가지만 추적:

```
docs_path:    docs/yyyyMMdd-기능요약/
branch:       feature/작업요약
workflow_ids: [id1, id2, ...]
ralf_cycle:   1
ralf_mode:    false   # true이면 3단계/4단계 완료 후 사용자 승인 단계를 건너뜀
```

---

## Tmux 인터랙티브 에이전트 디스패치 패턴

각 에이전트 단계는 아래 패턴으로 실행한다. **반드시 Tmux 세션 내에서 실행 중이어야 한다.**

### 원칙
- `claude -p`(헤드리스) 대신 `claude`(인터랙티브)를 사용한다
- 태스크는 파일로 미리 작성하고, `tmux send-keys`로 파일 경로를 전달한다
- 사용자는 Tmux 창을 전환하여 진행상황을 실시간으로 확인하고 필요 시 직접 인터랙션할 수 있다
- 에이전트 Claude는 작업 완료 시 **Bash 도구로 완료 신호 파일을 직접 기록**해야 한다

### 디스패치 절차

```bash
# 0. Tmux 세션 확인
if [ -z "$TMUX" ]; then
  echo "Error: n8n-studio는 Tmux 세션 내에서만 실행 가능합니다."
  exit 1
fi

# 1. 변수 설정
STAGE_NAME="[stage 이름 예: planner]"
WORK_DIR="$(pwd)"
TASK_DIR="/tmp/n8n-studio"
TASK_FILE="${TASK_DIR}/${STAGE_NAME}_task.md"
OUTPUT_FILE="${TASK_DIR}/${STAGE_NAME}_output.md"
STATUS_FILE="${TASK_DIR}/${STAGE_NAME}.done"
WINDOW_NAME="n8n-${STAGE_NAME}"

mkdir -p "$TASK_DIR"

# 2. 태스크 파일 작성
#    반드시 파일 마지막에 완료 신호 작성 지침을 포함한다 (아래 참고)
cat > "$TASK_FILE" << 'TASKEOF'
[태스크 내용 전체]

---
[작업 완료 후 반드시 실행]
모든 작업이 끝나면 Bash 도구로 아래 두 명령을 순서대로 실행하세요:
1. cat > /tmp/n8n-studio/[STAGE_NAME]_output.md << 'RESULTEOF'
   [결과 키-값, 예: docs_path: docs/20240410-xxx/]
   RESULTEOF
2. echo "DONE" > /tmp/n8n-studio/[STAGE_NAME].done
TASKEOF

# 3. 이전 상태 파일 제거
rm -f "$STATUS_FILE" "$OUTPUT_FILE"

# 4. 새 Tmux 창 생성 후 인터랙티브 Claude 시작
tmux new-window -n "$WINDOW_NAME"
tmux send-keys -t "$WINDOW_NAME" "cd '$WORK_DIR' && claude" Enter

# 5. Claude가 초기화될 때까지 대기 (프롬프트 출력 시간)
sleep 5

# 6. 태스크 파일 경로를 첫 메시지로 전송
tmux send-keys -t "$WINDOW_NAME" "다음 파일의 지침을 읽고 작업을 시작해주세요: $TASK_FILE" Enter

# 7. 사용자에게 Tmux 창 전환 안내 출력
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "  [$STAGE_NAME] Tmux 창 '$WINDOW_NAME' 에서 실행 중"
echo "  진행상황 확인: Ctrl+b → 창 번호 또는 Ctrl+b n"
echo "  완료 자동 감지 중 (폴링 10초 간격, 최대 30분)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

# 8. 완료 파일 폴링
TIMEOUT=1800
ELAPSED=0
while [ ! -f "$STATUS_FILE" ] && [ $ELAPSED -lt $TIMEOUT ]; do
  sleep 10
  ELAPSED=$((ELAPSED + 10))
done

if [ ! -f "$STATUS_FILE" ]; then
  echo "TIMEOUT: ${STAGE_NAME} 단계가 30분 내에 완료되지 않았습니다."
  exit 1
fi

# 9. 결과 파일 읽기
cat "$OUTPUT_FILE"
```

> **참고:** 결과 파일에서 `docs_path:`, `branch:`, `workflow_ids:`, `all_passed:` 등의 키를 파싱하여 오케스트레이터 상태를 업데이트한다.

---

## 1단계: 요청 유형 분류 (직접 처리)

사용자 요청을 분석하여 분류한다:
- **신규 개발**: 새 워크플로우 생성
- **기능 변경/추가**: 기존 워크플로우 수정
- **버그 수정**: 오류 또는 의도치 않은 동작 수정

## 2단계: 기본 정보 수집 (직접 처리 — 최소 인터랙션)

사용자에게 아래 정보만 질문한다:
- 관련 n8n 워크플로우 이름 또는 ID (복수 가능)
- 버그 수정 요청인 경우: Execution ID 유무

n8n-mcp로 지정된 워크플로우를 조회하여 ID를 확인한다. 연관 워크플로우가 있다고 판단되면 함께 조회한다.

> **요구사항 명확화는 여기서 하지 않는다.** 3단계 Tmux 창의 Claude가 직접 사용자와 인터랙션하며 요구사항을 구체화한다.

---

## 3단계: workflow-planner Tmux 에이전트 디스패치

Tmux 인터랙티브 디스패치 패턴으로 `n8n-planner` 창을 열고 아래 태스크를 전달한다.

**태스크 파일 내용 (`/tmp/n8n-studio/planner_task.md`):**

```
당신은 n8n 워크플로우 플래닝 전문가입니다.
workflow-planner 에이전트로서 아래 정보를 바탕으로 명세를 작성하세요.

[입력 정보]
- 요청 유형: [신규 개발 / 기능 변경 / 버그 수정]
- 대상 워크플로우: [이름: ID 목록]
- Execution ID: [있으면 기재, 없으면 없음]
- 사용자 요청 원문: [사용자가 입력한 원문 그대로]
- 진입 유형: [초기 진입 / 재진입 (실패 시나리오: ...) / 수정 재진입 (수정 지시: ...)]

[작업]
1. analyze-workflow 스킬로 워크플로우를 분석하세요
2. [초기 진입인 경우] 요구사항 중 불명확하거나 구체적이지 않은 사항을 파악하고
   사용자에게 직접 질문하여 요구사항을 구체화하세요.
   (사용자가 이 Tmux 창에서 직접 답변합니다)
   [재진입/수정 재진입인 경우] 사용자 질문 없이 이슈 기반으로 바로 진행하세요
3. write-spec 스킬로 docs/yyyyMMdd-기능요약/spec.md를 작성하세요
   (재진입/수정 재진입의 경우 기존 spec.md를 기반으로 수정하세요)
4. git checkout -b feature/작업요약 으로 브랜치를 생성하세요
   (이미 브랜치가 있으면 생략하세요)

[완료 후 반드시 실행]
모든 작업이 끝나면 Bash 도구로 아래 두 명령을 순서대로 실행하세요:
1. 결과 파일 기록:
   echo "docs_path: docs/[생성한 폴더명]/" > /tmp/n8n-studio/planner_output.md
   echo "branch: feature/[브랜치명]" >> /tmp/n8n-studio/planner_output.md
2. 완료 신호 기록:
   echo "DONE" > /tmp/n8n-studio/planner.done
```

오케스트레이터는 `/tmp/n8n-studio/planner.done` 파일을 폴링하여 완료를 감지한다.
완료 후 `/tmp/n8n-studio/planner_output.md`에서 `docs_path`와 `branch`를 파싱하여 상태에 저장한다.

**3단계 완료 후 사용자 검토 (ralf_mode=false인 경우에만):**

1. `[docs_path]/spec.md` 파일을 읽어 내용을 현재 세션에서 사용자에게 보여준다
2. 사용자에게 묻는다:
   ```
   spec.md 검토를 요청합니다.
   수정이 필요한 부분이 있으면 말씀해 주세요.
   없으시면 "승인"이라고 입력해 주세요.
   ```
3. 사용자가 승인하면 4단계로 진행한다
4. 사용자가 수정을 요청하면:
   - 수정 지시를 파악하고 3단계를 **수정 재진입** 모드로 재실행한다
   - 재실행 완료 후 다시 spec.md를 보여주고 승인을 요청한다
   - 사용자가 승인할 때까지 반복한다

---

## 4단계: workflow-designer Tmux 에이전트 디스패치

Tmux 인터랙티브 디스패치 패턴으로 `n8n-designer` 창을 열고 아래 태스크를 전달한다.

**태스크 파일 내용 (`/tmp/n8n-studio/designer_task.md`):**

```
당신은 n8n 워크플로우 설계 전문가입니다.
workflow-designer 에이전트로서 아래 파일을 읽고 설계를 수행하세요.

[입력 파일]
- 명세 문서: [docs_path]/spec.md
[수정 재진입의 경우 추가]
- 수정 지시: [사용자가 요청한 수정 내용]

[작업]
1. spec.md를 읽으세요
2. write-design 스킬로 [docs_path]/design.md를 작성하세요
   (수정 재진입의 경우 기존 design.md를 기반으로 수정하세요)
3. plan-tests 스킬로 테스트 파일을 작성하세요:
   - workflow/test/unit/ (단위 테스트)
   - workflow/test/acceptance/ (인수 테스트 JSON)
   - [docs_path]/test-plan.md (테스트 계획)

[완료 후 반드시 실행]
모든 작업이 끝나면 Bash 도구로 아래 두 명령을 순서대로 실행하세요:
1. 결과 파일 기록:
   echo "design_done: true" > /tmp/n8n-studio/designer_output.md
2. 완료 신호 기록:
   echo "DONE" > /tmp/n8n-studio/designer.done
```

**4단계 완료 후 사용자 검토 (ralf_mode=false인 경우에만):**

1. `[docs_path]/design.md` 파일을 읽어 내용을 현재 세션에서 사용자에게 보여준다
2. 사용자에게 묻는다:
   ```
   design.md 검토를 요청합니다.
   수정이 필요한 부분이 있으면 말씀해 주세요.
   없으시면 "승인"이라고 입력해 주세요.
   ```
3. 사용자가 승인하면 5단계로 진행한다
4. 사용자가 수정을 요청하면:
   - 수정 지시를 파악하고 4단계를 **수정 재진입** 모드로 재실행한다
   - 재실행 완료 후 다시 design.md를 보여주고 승인을 요청한다
   - 사용자가 승인할 때까지 반복한다

---

## 5단계: workflow-developer Tmux 에이전트 디스패치

Tmux 인터랙티브 디스패치 패턴으로 `n8n-developer` 창을 열고 아래 태스크를 전달한다.

**태스크 파일 내용 (`/tmp/n8n-studio/developer_task.md`):**

```
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

[완료 후 반드시 실행]
모든 작업이 끝나면 Bash 도구로 아래 두 명령을 순서대로 실행하세요:
1. 결과 파일 기록:
   echo "dev_done: true" > /tmp/n8n-studio/developer_output.md
   echo "workflow_ids: [실제 워크플로우 ID 목록, 쉼표 구분]" >> /tmp/n8n-studio/developer_output.md
   echo "refactored: [true/false]" >> /tmp/n8n-studio/developer_output.md
2. 완료 신호 기록:
   echo "DONE" > /tmp/n8n-studio/developer.done
```

완료 후 `/tmp/n8n-studio/developer_output.md`에서 `workflow_ids`를 파싱하여 상태에 저장한다.

---

## 7단계: workflow-verifier Tmux 에이전트 디스패치 (RALF 루프)

`ralf_cycle` 카운터를 초기화(1)하고 아래 루프를 수행한다.

Tmux 인터랙티브 디스패치 패턴으로 `n8n-verifier` 창을 열고 아래 태스크를 전달한다.

**태스크 파일 내용 (`/tmp/n8n-studio/verifier_task.md`):**

```
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

[완료 후 반드시 실행]
모든 작업이 끝나면 Bash 도구로 아래 두 명령을 순서대로 실행하세요:
1. 결과 파일 기록:
   echo "all_passed: [true/false]" > /tmp/n8n-studio/verifier_output.md
   echo "failed_scenarios: [실패 시나리오 ID 목록, 없으면 빈 배열]" >> /tmp/n8n-studio/verifier_output.md
   echo "failure_summary: [실패 원인 요약]" >> /tmp/n8n-studio/verifier_output.md
2. 완료 신호 기록:
   echo "DONE" > /tmp/n8n-studio/verifier.done
```

**결과 처리:**

- `all_passed: true` → 8단계로 진행
- `all_passed: false` AND `ralf_cycle < 3`:
  1. 실패 목록을 사용자에게 표시
  2. "재시도합니다 (사이클 [N]/3)" 알림
  3. `ralf_cycle += 1`, **`ralf_mode = true`** 로 설정
  4. 3단계로 돌아가 workflow-planner를 **재진입** 모드로 디스패치 (실패 시나리오 전달)
  5. 4단계 → 5단계 → 7단계 반복
- `all_passed: false` AND `ralf_cycle == 3`:
  `ralf_mode = false`로 초기화 후 보고하고 사용자 지침 대기:
  ```
  ## 검증 실패 보고 (사이클 3/3)
  ### 실패한 시나리오
  [failure_summary 내용]
  사용자 지침이 필요합니다. 어떻게 진행할까요?
  ```
  지침 수신 후 `ralf_cycle = 1`로 초기화하여 3단계부터 재시작

---

## 8단계: workflow-finisher Tmux 에이전트 디스패치

Tmux 인터랙티브 디스패치 패턴으로 `n8n-finisher` 창을 열고 아래 태스크를 전달한다.

**태스크 파일 내용 (`/tmp/n8n-studio/finisher_task.md`):**

```
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

[완료 후 반드시 실행]
모든 작업이 끝나면 Bash 도구로 아래 두 명령을 순서대로 실행하세요:
1. echo "[PR URL]" > /tmp/n8n-studio/finisher_output.md
2. echo "DONE" > /tmp/n8n-studio/finisher.done
```

완료 후 `/tmp/n8n-studio/finisher_output.md`에서 PR URL을 읽어 사용자에게 보고하며 완료한다.

---

## 중단 조건

아래 상황에서만 오케스트레이터(현재 세션)가 사용자 입력을 기다린다:
- **2단계**: 워크플로우 ID 질문
- **3단계 완료 후** (ralf_mode=false): 사용자 spec.md 검토 및 승인
- **4단계 완료 후** (ralf_mode=false): 사용자 design.md 검토 및 승인
- **7단계**: 3회 사이클 실패 후 지침 대기
- 그 외는 멈추지 않고 자동 진행한다

> **참고:** 3단계 진행 중 요구사항 인터뷰는 `n8n-planner` Tmux 창에서 사용자가 직접 인터랙션한다. 오케스트레이터는 완료 신호 파일만 기다린다.
