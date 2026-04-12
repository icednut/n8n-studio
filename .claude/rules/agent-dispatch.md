# Agent Dispatch Rules: Team Lead ↔ 서브에이전트 (Tmux 기반)

## 적용 범위

n8n-studio:start 스킬이 실행 중인 **Team Lead 세션**에 적용된다.
서브에이전트 pane에서 실행 중인 Claude 세션에는 적용되지 않는다.

---

## 규칙 1: 3단계 이후 작업은 반드시 전담 pane에 디스패치한다

Team Lead는 3단계(planning) 이후 어떤 작업도 자신의 세션에서 직접 실행하지 않는다.
반드시 아래 전담 pane에 태스크 파일을 작성하고 tmux send-keys로 전달한다.

| 단계 | 전담 Pane | 모델 |
|------|----------|------|
| 3단계 (planning) | n8n-planner | haiku |
| 4단계 (design) | n8n-designer | sonnet |
| 5·6단계 (develop + refactor) | n8n-developer | sonnet |
| 7단계 (verify) | n8n-verifier | haiku |
| 8단계 (finish) | n8n-finisher | haiku |

**위반 예시 (금지):**
- Team Lead 세션에서 직접 `analyze-workflow` 스킬을 실행
- Team Lead 세션에서 직접 `write-spec` 스킬을 실행
- Agent tool로 서브에이전트를 직접 호출

---

## 규칙 2: 서브에이전트 완료는 push 알림으로만 수신한다

Team Lead는 완료 여부를 파일 폴링이나 능동적 조회로 확인하지 않는다.
서브에이전트는 작업 완료 시 반드시 아래 방식으로 Team Lead에게 보고한다:

```bash
tmux send-keys -t [TEAM_LEAD_PANE_ID] "[보고 메시지]" Enter
```

Team Lead는 이 메시지를 수신할 때까지 대기 상태를 유지한다.

---

## 규칙 2-1: 모든 서브에이전트는 작업 완료 시 반드시 Team Lead에게 보고한다

**이 규칙은 모든 서브에이전트(n8n-planner, n8n-designer, n8n-developer, n8n-verifier, n8n-finisher)에 예외 없이 적용된다.**

### 보고 의무

서브에이전트는 맡은 작업이 완료되는 즉시, 아래 두 단계를 반드시 수행한다:

1. **결과 파일 기록**: `/tmp/n8n-studio/[stage]_output.md` 에 결과 키-값을 저장한다.
2. **Team Lead에게 tmux 보고**: 태스크 파일에 명시된 `TEAM_LEAD_PANE_ID`로 완료 메시지를 전송한다.

```bash
tmux send-keys -t [TEAM_LEAD_PANE_ID] "✅ [에이전트명] 완료. [결과 요약]. 다음 단계를 진행해주세요." Enter
```

### 보고 금지 패턴 (위반 예시)

- 작업 완료 후 그냥 응답을 멈추는 행위
- 결과 파일만 저장하고 tmux 보고를 생략하는 행위
- "완료했습니다"라고 자기 pane에만 출력하고 Team Lead에게 전송하지 않는 행위

### 보고가 누락되면 발생하는 문제

Team Lead는 서브에이전트의 tmux 보고를 수신하기 전까지 다음 단계를 진행하지 않는다.
보고가 누락되면 전체 개발 사이클이 해당 단계에서 멈춘다.

**서브에이전트는 tmux 보고를 작업의 마지막 필수 단계로 간주해야 한다.**

---

## 규칙 3: RALF 루프는 전담 pane 시퀀스를 그대로 따른다

검증 실패 시 RALF 재진입은 반드시 아래 순서로 각 전담 pane에 태스크를 재전송한다:

```
n8n-planner → n8n-designer → n8n-developer → n8n-verifier
```

어떤 단계도 건너뛰거나 Team Lead 또는 n8n-verifier 세션에서 직접 처리하지 않는다.

---

## 규칙 4: Pane은 Team Lead 윈도우에만 생성한다

```bash
# 올바른 방법
tmux split-window -h -d -t "$TEAM_LEAD_WINDOW" -P -F "#{pane_id}"

# 금지 (현재 포커스된 윈도우에 생성됨)
tmux split-window -h -d -P -F "#{pane_id}"
```

---

## 규칙 5: 기존 pane은 재사용한다

동일 이름의 pane이 살아있으면 새로 생성하지 않고 기존 pane에 새 태스크를 전송한다.

```bash
EXISTING_PANE=$(tmux list-panes -t "$TEAM_LEAD_WINDOW" -F "#{pane_id} #{pane_title}" \
  | grep " ${PANE_NAME}$" | awk '{print $1}' | head -1)
```

---

## 규칙 6: Pane 생성 즉시 이름 부여 및 tiled 레이아웃 적용

```bash
PANE_ID=$(tmux split-window -h -d -t "$TEAM_LEAD_WINDOW" -P -F "#{pane_id}")
tmux select-pane -t "$PANE_ID" -T "$PANE_NAME"
tmux select-layout -t "$TEAM_LEAD_WINDOW" tiled
```

---

## 규칙 7: Claude Code 실행 확인 후 프롬프트를 전송한다

신규 pane에서 `claude` 명령을 실행한 뒤, Claude Code의 프롬프트(`:` 또는 `>`)가 출력되어 입력 대기 상태임이 확인되기 전까지 태스크 파일 경로를 전송하지 않는다.

**확인 방법:**

```bash
# claude 실행
tmux send-keys -t "$PANE_ID" "cd '$WORK_DIR' && claude --model $PANE_MODEL --name '$PANE_NAME'" Enter

# Claude Code 프롬프트가 나타날 때까지 대기 (최대 30초, 1초 간격 폴링)
for i in $(seq 1 30); do
  PANE_CONTENT=$(tmux capture-pane -t "$PANE_ID" -p 2>/dev/null)
  if echo "$PANE_CONTENT" | grep -qE '(^|\s)(>|✓|✗|\?)(\s|$)'; then
    break
  fi
  sleep 1
done
```

**규칙:**
- 루프가 30초를 초과하여 종료되면 해당 pane에 태스크를 전송하지 않고 Team Lead는 사용자에게 오류를 보고한다.
- 기존 pane을 재사용하는 경우(규칙 5)에도 동일하게 프롬프트 대기 상태를 확인한 후 전송한다.
