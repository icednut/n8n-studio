---
name: n8n-studio:run-verification
description: This skill should be used by n8n-studio agents when they need to execute all integration test scenarios and evaluate the results. Runs n8n Test Execution for each scenario, compares actual vs expected output, and returns a pass/fail report.
---

# run-verification

단일 책임: 통합 테스트 시나리오를 실행하고 기대값과 실제값을 비교하여 결과 보고서를 생성한다.

## 수행 작업

### 테스트 파일 로드

`workflow/[프로젝트명]/test/integration/` 의 모든 JSON 파일을 읽어 실행할 시나리오 목록 수집.

### 시나리오별 실행

각 시나리오에 대해:

1. **테스트 데이터 준비**

   시나리오 JSON의 `input.data`를 Test Execution 입력으로 사용한다.
   해당 워크플로우의 `workflow/[프로젝트명]/summary/[workflowId].json`에 `triggerDataFormat`이 있는 경우:
   - `input.data`가 `triggerDataFormat` 기반으로 작성되었는지 확인
   - 시나리오별로 필드 값이 적절히 변형되어 있는지 확인 (구조/타입 일관성 검증)

2. **n8n Test Execution 실행**
   - `n8n_test_workflow` 도구 사용
   - 시나리오 JSON의 `input.data`를 입력 파라미터로 전달
   - 대상 워크플로우: `input.workflowId`

3. **결과 수집**
   - 실행 완료 후 출력 데이터 수집
   - 실행 로그 확인 (오류 여부)
   - Execution ID 기록 (이후 분석에 활용 가능)

4. **기대값 비교**

   `expectedOutput.type`에 따라 다르게 검증:

   - **`json` 타입**: 실제 출력 JSON과 `expectedOutput.value` 비교
     - 구조 및 값이 일치하는지 확인

5. **서브 워크플로우 호출 검증** (`subWorkflowVerification` 있는 경우)

   시나리오에 `subWorkflowVerification` 항목이 있는 경우, 해당 노드의 실행 여부와 호출 파라미터를 검증한다.
   B 워크플로우의 실제 실행 결과는 검증하지 않는다.

   검증 항목:
   - 해당 `nodeName`의 "Execute Workflow" 노드가 실행(완료 또는 오류 없이 호출)되었는지 확인
   - 노드의 입력 데이터가 `expectedInputData`와 일치하는지 확인

   ```
   서브 워크플로우 호출 검증:
   - 노드: [nodeName] → 워크플로우 ID: [calledWorkflowId]
   - 호출 여부: ✅ 호출됨 / ❌ 호출되지 않음
   - 입력 데이터 일치: ✅ 일치 / ❌ 불일치 (기대: {...} / 실제: {...})
   ```

6. **결과 기록**
   - 통과: 시나리오 ID, 설명
   - 실패: 시나리오 ID, 설명, 기대값, 실제값, 실패 원인

### 결과 보고서 형식

```
## 검증 결과 (사이클 N)

### 통과 ✅ (N개)
- IT-001: [시나리오 설명]
- IT-002: [시나리오 설명]

### 실패 ❌ (N개)
| 시나리오 | 기대값 | 실제값 | 원인 |
|---------|--------|--------|------|
| IT-003  | {...}  | {...}  | [오류 메시지] |

### 서브 워크플로우 호출 검증
| 시나리오 | 노드 | 호출 여부 | 입력 데이터 일치 |
|---------|------|---------|--------------|
| IT-001  | [노드명] | ✅ | ✅ |

### 요약
- 전체: N개 / 통과: N개 / 실패: N개
```

실패가 없으면 → 전체 통과 보고
실패가 있으면 → 실패 목록과 함께 보고
