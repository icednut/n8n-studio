---
name: n8n-studio:start
description: This skill should be used when the user asks to "n8n 워크플로우 개발해줘", "n8n 워크플로우 만들어줘", "n8n 워크플로우 기능 추가해줘", "n8n 워크플로우 수정해줘", "n8n 워크플로우 버그 수정해줘", "n8n 워크플로우 고쳐줘", or any request related to creating, modifying, or fixing n8n workflows. This is the main entry point and full orchestrator for the n8n-studio development cycle.
argument-hint: "[개발/수정/버그수정 요청 설명]"
allowed-tools: ["Read", "Write", "Bash", "Glob", "Grep", "mcp__n8n-mcp__n8n_list_workflows", "mcp__n8n-mcp__n8n_get_workflow", "mcp__n8n-mcp__n8n_executions"]
---

# n8n-studio:start (Team Lead)

전체 8단계 개발 사이클의 **Team Lead**. 1·2단계를 직접 처리하고, 3단계부터는 각 서브에이전트를 **이름이 부여된 Tmux Pane**에서 실행한다. 서브에이전트가 완료되면 **tmux send-keys로 Team Lead에게 직접 보고**하며, Team Lead는 폴링 없이 보고를 받는 즉시 다음 단계를 진행한다.

## Team Lead의 역할

- **1·2단계 직접 처리**: 요청 분류 및 기본 정보 수집
- **서브에이전트 디스패치**: 3~8단계를 각 전담 pane으로 실행
- **사용자 지침 릴레이**: 사용자가 특정 서브에이전트에 지침을 주면 해당 pane으로 전달
- **상태 추적**: docs_path, branch, workflow_ids, ralf_cycle, ralf_mode 유지

## 커뮤니케이션 원칙

- **간결하고 사무적인 말투**: 불필요한 수식어, 감탄사, 격려 문구 금지
- **동의는 사실 기반으로만**: 사용자 말이 옳을 때만 동의한다. 무조건적인 "네, 맞습니다", "좋은 생각이에요" 금지
- **상태 업데이트 형식**: `[N단계] [에이전트명] 디스패치 완료. [pane명]에서 실행 중.` 형태로 통일
- **질문 형식**: 한 번에 하나의 질문만. 선택지는 번호 목록으로 제시
- **보고 형식**: `완료: [결과 요약].` 또는 `실패: [원인]. [다음 액션].`

## 오케스트레이터 상태

```
docs_path:    docs/yyyyMMdd-기능요약/
branch:       feature/작업요약
workflow_ids: [id1, id2, ...]
ralf_cycle:   1
ralf_mode:    false
```

---

## 초기화 (스킬 시작 시 1회 실행)

```bash
# Tmux 세션 확인
if [ -z "$TMUX" ]; then
  echo "Error: n8n-studio는 Tmux 세션 내에서만 실행 가능합니다."
  exit 1
fi

# Team Lead 자신의 pane ID 및 window ID 캡처 및 저장
mkdir -p /tmp/n8n-studio
TEAM_LEAD_PANE=$(tmux display-message -p "#{pane_id}")
TEAM_LEAD_WINDOW=$(tmux display-message -p "#{window_id}")
echo "$TEAM_LEAD_PANE" > /tmp/n8n-studio/team_lead_pane.txt
echo "$TEAM_LEAD_WINDOW" > /tmp/n8n-studio/team_lead_window.txt
echo "Team Lead pane: $TEAM_LEAD_PANE (window: $TEAM_LEAD_WINDOW)"
```

---

## Pane 관리 패턴

모든 서브에이전트 디스패치에서 아래 패턴을 사용한다.

### 원칙

- **Pane 생성 즉시 이름 부여**: `tmux select-pane -t "$PANE_ID" -T "$PANE_NAME"`
- **기존 pane 재사용**: 동일 이름의 pane이 살아있으면 새로 생성하지 않고 재사용
- **완료 후 pane 유지**: 서브에이전트가 작업을 마쳐도 pane을 닫지 않음
- **Push 알림**: 서브에이전트가 완료 시 `tmux send-keys`로 Team Lead pane에 직접 보고

### 디스패치 절차

```bash
WORK_DIR="$(pwd)"
TASK_DIR="/tmp/n8n-studio"
PANE_NAME="[pane 이름 예: n8n-planner]"
PANE_MODEL="[에이전트 모델 예: haiku 또는 sonnet]"
TASK_FILE="${TASK_DIR}/[stage]_task.md"
TEAM_LEAD_PANE=$(cat /tmp/n8n-studio/team_lead_pane.txt)
TEAM_LEAD_WINDOW=$(cat /tmp/n8n-studio/team_lead_window.txt)

# 1. 태스크 파일 작성 (완료 보고 명령 포함)
cat > "$TASK_FILE" << 'TASKEOF'
[태스크 내용]

---
## 완료 보고 (작업 완료 후 반드시 실행)
모든 작업이 끝나면 Bash 도구로 아래 명령을 순서대로 실행하세요:

1. 결과 파일 기록:
   cat > /tmp/n8n-studio/[stage]_output.md << 'RESULTEOF'
   [결과 키-값]
   RESULTEOF

2. Team Lead에게 보고:
   tmux send-keys -t [TEAM_LEAD_PANE_ID] "[stage] 완료. [결과 요약]. 다음 단계를 진행해주세요." Enter
TASKEOF

# 2. 기존 pane 조회 (team-lead 윈도우 내에서만)
EXISTING_PANE=$(tmux list-panes -t "$TEAM_LEAD_WINDOW" -F "#{pane_id} #{pane_title}" 2>/dev/null \
  | grep " ${PANE_NAME}$" | awk '{print $1}' | head -1)

if [ -n "$EXISTING_PANE" ]; then
  # 3a. 기존 pane 재사용
  PANE_ID="$EXISTING_PANE"
else
  # 3b. team-lead 윈도우에 새 pane 생성 → 즉시 이름 부여 → 타일 재배치 → claude 시작
  PANE_ID=$(tmux split-window -h -d -t "$TEAM_LEAD_WINDOW" -P -F "#{pane_id}")
  tmux select-pane -t "$PANE_ID" -T "$PANE_NAME"
  tmux select-layout -t "$TEAM_LEAD_WINDOW" tiled
  tmux send-keys -t "$PANE_ID" "cd '$WORK_DIR' && claude --model $PANE_MODEL" Enter
  sleep 5
fi

# 4. 태스크 파일 경로 전송
tmux send-keys -t "$PANE_ID" "다음 파일의 지침을 읽고 작업을 시작해주세요: $TASK_FILE" Enter
```

> **Team Lead는 이 시점에서 대기 상태로 전환한다.** 서브에이전트가 완료되면 tmux send-keys로 메시지를 보내오며, 그 메시지를 받는 즉시 다음 단계를 진행한다.

---

## 사용자 지침 릴레이

사용자가 작업 중인 서브에이전트에 지침을 전달하고 싶다고 말하면, Team Lead는 해당 pane을 찾아 메시지를 전달한다.

```bash
# 예: 사용자가 "planner에게 xxx를 알려줘"라고 하면
TARGET_PANE=$(tmux list-panes -s -F "#{pane_id} #{pane_title}" 2>/dev/null \
  | grep " n8n-planner$" | awk '{print $1}' | head -1)

if [ -n "$TARGET_PANE" ]; then
  tmux send-keys -t "$TARGET_PANE" "사용자 지침: [지침 내용]" Enter
else
  echo "n8n-planner pane을 찾을 수 없습니다."
fi
```

---

## 1단계: 요청 유형 분류 (Team Lead 직접 처리)

사용자 요청을 분석하여 분류한다:
- **신규 개발**: 새 워크플로우 생성
- **기능 변경/추가**: 기존 워크플로우 수정
- **버그 수정**: 오류 또는 의도치 않은 동작 수정

---

## 2단계: 기본 정보 수집 (Team Lead 직접 처리 — 최소 인터랙션)

사용자에게 아래 정보만 질문한다:
- 관련 n8n 워크플로우 이름 또는 ID (복수 가능)
- 버그 수정 요청인 경우: Execution ID 유무

n8n-mcp로 지정된 워크플로우를 조회하여 ID를 확인한다. 연관 워크플로우가 있다고 판단되면 함께 조회한다.

> **요구사항 명확화는 여기서 하지 않는다.** 3단계 `n8n-planner` pane의 Claude가 직접 사용자와 인터랙션하며 요구사항을 구체화한다.

---

## 3단계: workflow-planner 디스패치 (Pane: n8n-planner, Model: haiku)

**태스크 파일 (`/tmp/n8n-studio/planner_task.md`):**

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
2. [초기 진입인 경우] 불명확한 요구사항을 사용자에게 직접 질문하여 구체화하세요
   [재진입/수정 재진입인 경우] 사용자 질문 없이 이슈 기반으로 바로 진행하세요
3. write-spec 스킬로 docs/yyyyMMdd-기능요약/01-spec.md를 작성하세요
4. git checkout -b feature/작업요약 으로 브랜치를 생성하세요 (이미 있으면 생략)

---
## 완료 보고 (작업 완료 후 반드시 실행)
모든 작업이 끝나면 Bash 도구로 아래 명령을 순서대로 실행하세요:

1. 결과 파일 기록:
   echo "docs_path: docs/[생성한 폴더명]/" > /tmp/n8n-studio/planner_output.md
   echo "branch: feature/[브랜치명]" >> /tmp/n8n-studio/planner_output.md

2. Team Lead에게 보고:
   tmux send-keys -t [TEAM_LEAD_PANE_ID] "✅ [n8n-planner] 완료. docs_path=docs/[폴더명]/, branch=feature/[브랜치명]. 다음 단계를 진행해주세요." Enter
```

완료 보고를 받으면 `/tmp/n8n-studio/planner_output.md`에서 `docs_path`와 `branch`를 파싱하여 상태에 저장한다.

**3단계 완료 후 사용자 검토 (ralf_mode=false인 경우에만):**

1. `[docs_path]/01-spec.md`를 읽어 사용자에게 보여준다
2. 승인 또는 수정 요청을 받는다
3. 수정 요청 시: `n8n-planner` pane에 **수정 재진입** 태스크를 재전송한다
4. 승인 시 4단계로 진행한다

---

## 4단계: workflow-designer 디스패치 (Pane: n8n-designer, Model: sonnet)

**태스크 파일 (`/tmp/n8n-studio/designer_task.md`):**

```
당신은 n8n 워크플로우 설계 전문가입니다.
workflow-designer 에이전트로서 아래 파일을 읽고 설계를 수행하세요.

[입력 파일]
- 명세 문서: [docs_path]/01-spec.md
[수정 재진입의 경우 추가]
- 수정 지시: [사용자가 요청한 수정 내용]

[작업]
1. 01-spec.md를 읽으세요
2. write-design 스킬로 [docs_path]/02-design.md를 작성하세요
3. plan-tests 스킬로 테스트 파일을 작성하세요:
   - workflow/test/unit/ (단위 테스트)
   - workflow/test/acceptance/ (인수 테스트 JSON)
   - [docs_path]/03-test-plan.md (테스트 계획)

---
## 완료 보고 (작업 완료 후 반드시 실행)
모든 작업이 끝나면 Bash 도구로 아래 명령을 순서대로 실행하세요:

1. 결과 파일 기록:
   echo "design_done: true" > /tmp/n8n-studio/designer_output.md

2. Team Lead에게 보고:
   tmux send-keys -t [TEAM_LEAD_PANE_ID] "✅ [n8n-designer] 완료. 02-design.md 및 테스트 시나리오 작성 완료. 다음 단계를 진행해주세요." Enter
```

**4단계 완료 후 사용자 검토 (ralf_mode=false인 경우에만):**

1. `[docs_path]/02-design.md`를 읽어 사용자에게 보여준다
2. 승인 또는 수정 요청을 받는다
3. 수정 요청 시: `n8n-designer` pane에 **수정 재진입** 태스크를 재전송한다
4. 승인 시 5단계로 진행한다

---

## 5·6단계: workflow-developer 디스패치 (Pane: n8n-developer, Model: sonnet)

개발(5단계)과 리팩토링(6단계)을 동일한 `n8n-developer` pane에서 연속으로 처리한다.

**태스크 파일 (`/tmp/n8n-studio/developer_task.md`):**

```
당신은 n8n 워크플로우 개발 전문가입니다.
workflow-developer 에이전트로서 아래 파일을 읽고 개발과 리팩토링을 수행하세요.

[입력 파일]
- 설계 문서: [docs_path]/02-design.md

[작업]
1. 02-design.md를 읽으세요
2. Code 노드가 필요하면 develop-code-node 스킬로 TDD 개발하세요
3. build-workflow 스킬로 n8n 워크플로우를 생성/수정하세요
4. refactor 스킬로 리팩토링 필요 여부를 판단하고, 필요하면 진행하세요
   (필요 없다고 판단되면 건너뜁니다)

---
## 완료 보고 (모든 작업 완료 후 반드시 실행)
모든 작업이 끝나면 Bash 도구로 아래 명령을 순서대로 실행하세요:

1. 결과 파일 기록:
   echo "dev_done: true" > /tmp/n8n-studio/developer_output.md
   echo "workflow_ids: [실제 워크플로우 ID 목록, 쉼표 구분]" >> /tmp/n8n-studio/developer_output.md
   echo "refactored: [true/false]" >> /tmp/n8n-studio/developer_output.md

2. Team Lead에게 보고:
   tmux send-keys -t [TEAM_LEAD_PANE_ID] "✅ [n8n-developer] 완료. workflow_ids=[ID목록], refactored=[true/false]. 다음 단계를 진행해주세요." Enter
```

완료 보고를 받으면 `/tmp/n8n-studio/developer_output.md`에서 `workflow_ids`를 파싱하여 상태에 저장한다.

---

## 7단계: workflow-verifier 디스패치 (Pane: n8n-verifier, Model: haiku, RALF 루프)

`ralf_cycle` 카운터를 초기화(1)하고 아래 루프를 수행한다.

**태스크 파일 (`/tmp/n8n-studio/verifier_task.md`):**

```
당신은 n8n 워크플로우 검증 전문가입니다.
workflow-verifier 에이전트로서 통합 테스트를 실행하세요.

[입력]
- 테스트 계획: [docs_path]/03-test-plan.md
- 테스트 파일 경로: workflow/test/acceptance/
- 대상 워크플로우 ID: [workflow_ids]
- 현재 사이클: [ralf_cycle]/3

[작업]
1. run-verification 스킬로 모든 시나리오를 처음부터 실행하세요
2. 결과를 판단하세요

---
## 완료 보고 (작업 완료 후 반드시 실행)
모든 작업이 끝나면 Bash 도구로 아래 명령을 순서대로 실행하세요:

1. 결과 파일 기록:
   echo "all_passed: [true/false]" > /tmp/n8n-studio/verifier_output.md
   echo "failed_scenarios: [실패 시나리오 ID 목록, 없으면 빈 배열]" >> /tmp/n8n-studio/verifier_output.md
   echo "failure_summary: [실패 원인 요약]" >> /tmp/n8n-studio/verifier_output.md

2. Team Lead에게 보고:
   tmux send-keys -t [TEAM_LEAD_PANE_ID] "✅ [n8n-verifier] 완료. all_passed=[true/false]. 다음 단계를 진행해주세요." Enter
```

**결과 처리:**

- `all_passed: true` → 8단계로 진행
- `all_passed: false` AND `ralf_cycle < 3`:
  1. 실패 목록을 사용자에게 표시
  2. "재시도합니다 (사이클 [N]/3)" 알림
  3. `ralf_cycle += 1`, `ralf_mode = true` 로 설정
  4. 3단계로 돌아가 `n8n-planner` pane에 **재진입** 태스크 재전송
  5. 4단계 → 5단계 → 6단계 → 7단계 반복
- `all_passed: false` AND `ralf_cycle == 3`:
  `ralf_mode = false`로 초기화 후 사용자 지침 대기:
  ```
  ## 검증 실패 보고 (사이클 3/3)
  ### 실패한 시나리오
  [failure_summary 내용]
  사용자 지침이 필요합니다. 어떻게 진행할까요?
  ```
  지침 수신 후 `ralf_cycle = 1`로 초기화하여 3단계부터 재시작

---

## 8단계: workflow-finisher 디스패치 (Pane: n8n-finisher, Model: haiku)

**태스크 파일 (`/tmp/n8n-studio/finisher_task.md`):**

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
1. [docs_path]/04-result.md 작성
2. 워크플로우 JSON을 workflow/ 폴더에 저장
3. git add/commit/push
4. GitHub PR 생성 (base: main)

---
## 완료 보고 (작업 완료 후 반드시 실행)
모든 작업이 끝나면 Bash 도구로 아래 명령을 순서대로 실행하세요:

1. 결과 파일 기록:
   echo "[PR URL]" > /tmp/n8n-studio/finisher_output.md

2. Team Lead에게 보고:
   tmux send-keys -t [TEAM_LEAD_PANE_ID] "✅ [n8n-finisher] 완료. PR=[PR URL]. 작업이 모두 완료되었습니다." Enter
```

완료 보고를 받으면 `/tmp/n8n-studio/finisher_output.md`에서 PR URL을 읽어 사용자에게 최종 보고한다.

---

## 중단 조건 (Team Lead가 사용자 입력을 기다리는 시점)

- **2단계**: 워크플로우 ID 질문
- **3단계 완료 후** (ralf_mode=false): 01-spec.md 검토 및 승인 대기
- **4단계 완료 후** (ralf_mode=false): 02-design.md 검토 및 승인 대기
- **7단계**: 3회 사이클 실패 후 지침 대기
- **서브에이전트 작업 중**: 완료 보고 대기 (사용자 지침 릴레이는 가능)
- 그 외는 멈추지 않고 자동 진행한다
