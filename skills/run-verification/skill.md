---
name: n8n-studio:run-verification
description: This skill should be used by n8n-studio agents when they need to execute all integration test scenarios and evaluate the results. Runs n8n Test Execution for each scenario, compares actual vs expected output, and returns a pass/fail report.
---

# run-verification

단일 책임: 통합 테스트 시나리오를 실행하고 기대값과 실제값을 비교하여 결과 보고서를 생성한다.

## 수행 작업

### 테스트 파일 로드

`workflow/test/acceptance/` 의 모든 JSON 파일을 읽어 실행할 시나리오 목록 수집.

### 시나리오별 실행

각 시나리오에 대해:

1. **n8n Test Execution 실행**
   - `n8n_test_workflow` 도구 사용
   - 시나리오 JSON의 `input.data`를 입력 파라미터로 전달
   - 대상 워크플로우: `input.workflowId`

2. **결과 수집**
   - 실행 완료 후 출력 데이터 수집
   - 실행 로그 확인 (오류 여부)

3. **기대값 비교**

   `expectedOutput.type`에 따라 다르게 검증:

   - **`json` 타입**: 실제 출력 JSON과 `expectedOutput.value` 비교
     - 구조 및 값이 일치하는지 확인
   
   - **`file` 타입**: `expectedOutput.path`의 파일 내용 읽기
     - 파일 내용이 `expectedOutput.content`와 일치하는지 확인
     - 정규식 패턴인 경우 패턴 매칭으로 검증

4. **결과 기록**
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

### 요약
- 전체: N개 / 통과: N개 / 실패: N개
```

실패가 없으면 → 전체 통과 보고
실패가 있으면 → 실패 목록과 함께 보고
