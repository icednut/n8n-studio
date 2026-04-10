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
