---
name: n8n-studio:verify
description: This skill should be used when the n8n-studio development cycle reaches the verification phase (7단계), or when the user asks to "n8n 워크플로우 검증해줘", "통합 테스트 실행해줘". Runs integration test scenarios in a RALF loop (up to 3 cycles) and reports failures to the user if all cycles fail.
argument-hint: "[docs 폴더 경로 (선택)]"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep", "Agent", "mcp__n8n-mcp__n8n_test_workflow", "mcp__n8n-mcp__n8n_executions", "mcp__n8n-mcp__n8n_get_workflow"]
---

# n8n-studio:verify

7단계 검증을 담당한다. 설계 단계에서 작성한 통합 테스트 시나리오를 실행하고 RALF 루프로 자가 수정한다.

## RALF 루프 구조

```
Read (현황 파악) → Act (테스트 실행) → Learn (근본 원인 분석) → Fix (명세 재수립 후 재개발)
```

각 단계의 역할:
- **Read**: 테스트 시나리오와 기대값을 파악하고, 실행 로그/결과물을 수집한다
- **Act**: n8n Test Execution으로 전체 시나리오를 실행한다
- **Learn**: 실패 시나리오의 실제값 vs 기대값 차이를 분석하고, 실행 로그에서 근본 원인(root cause)을 파악한다. 코드 오류인지, 설계 오류인지, 테스트 데이터 오류인지 분류한다
- **Fix**: 파악된 근본 원인을 바탕으로 `start` 3단계로 재진입하여 이슈 해결 명세를 수립하고, 설계→개발→검증 사이클을 다시 수행한다

사이클은 최대 3회 반복한다. 3회 모두 실패 시 사용자에게 보고하고 지침을 기다린다.

## 처리 흐름

### 준비

1. `docs/yyyyMMdd-기능요약/03-test-plan.md` 읽기
2. `workflow/[프로젝트명]/test/acceptance/` 의 테스트 파일 목록 확인
3. 현재 사이클 번호 초기화 (1)

### 테스트 실행 (run-verification 스킬)

workflow-verifier 에이전트를 통해 `run-verification` 스킬로 전체 테스트 시나리오를 처음부터 실행한다.

**실행 방법:**
- n8n-mcp의 Test Execution 기능으로 각 시나리오 실행
- 시나리오별 입력 파라미터 JSON을 `workflow/[프로젝트명]/test/acceptance/` 파일에서 읽음
- 실행 결과 수집:
  - 출력값과 기대값 비교
  - 파일 산출물이 있는 경우 파일 내용 비교
  - 실행 로그로 오류 여부 판단

**실패 판단 기준:**
- 기대 출력값과 실제 출력값이 불일치
- 기대 파일 내용과 실제 파일 내용이 불일치
- 워크플로우 실행 중 오류 발생

### 결과 판단

**모두 통과한 경우:**
- 자동으로 `finish` 스킬로 전환

**실패한 시나리오가 있는 경우 (사이클 3회 미만):**
1. 실패 시나리오 목록을 사용자에게 보여줌
2. "재시도합니다" 알림 후 자동 진행
3. `start` 스킬(3단계 재진입)로 돌아가 이슈 기반 명세 재수립
4. `design` → `develop` → `verify` 사이클 반복
5. 사이클 카운터 +1

**3회 사이클 모두 실패한 경우:**
아래 형식으로 사용자에게 보고하고 지침을 기다린다:

```
## 검증 실패 보고 (사이클 3/3)

### 통과한 시나리오
- [목록]

### 실패한 시나리오
| 시나리오 | 기대값 | 실제값 | 실패 원인 |
|---------|--------|--------|---------|
| ...     | ...    | ...    | ...     |

사용자 지침이 필요합니다. 어떻게 진행할까요?
```

**사용자 지침 수신 후:**
- 사이클 카운터 초기화 (다시 3회 기회)
- `start` 스킬 3단계로 재진입하여 새 사이클 시작
- 이 반복은 횟수 제한 없이 계속됨

## 중단 조건

- 3회 사이클 실패 후 사용자 지침을 기다리는 경우만 멈춤
- 그 외 사이클 내 자동 진행 중에는 멈추지 않음
