---
name: n8n-studio:refactor
description: This skill should be used when the n8n-studio development cycle reaches the refactoring phase (6단계), or when the user asks to "n8n 워크플로우 리팩토링해줘", "워크플로우 단순화해줘". This phase is optional and can be skipped if not needed.
argument-hint: "[docs 폴더 경로 (선택)]"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep", "mcp__n8n-mcp__n8n_get_workflow", "mcp__n8n-mcp__n8n_update_full_workflow", "mcp__n8n-mcp__n8n_update_partial_workflow"]
---

# n8n-studio:refactor

6단계 리팩토링을 담당한다. 개발 완료된 n8n 워크플로우를 간결하고 유지보수하기 쉽게 개선한다. **이 단계는 선택적이며, 불필요하다고 판단되면 건너뛴다.**

workflow-developer 에이전트가 5단계 개발 완료 후 이 스킬을 직접 사용하여 진행한다.

## 리팩토링 판단 기준

아래 항목 중 하나라도 해당되면 리팩토링을 진행한다:

- 동일한 로직이 여러 노드에 중복되어 있음
- 단일 Code 노드가 너무 많은 역할을 담당함 (SRP 위반)
- 워크플로우 흐름을 한눈에 파악하기 어려울 정도로 복잡함
- 노드 이름이 역할을 명확히 나타내지 않음
- 불필요한 노드가 포함되어 있음

해당 항목이 없으면 이 단계를 건너뛰고 완료 보고를 진행한다.

## 리팩토링 처리 흐름

1. n8n-mcp로 현재 워크플로우 구조 확인
2. 리팩토링 대상 식별 (위 판단 기준 기반)
3. 개선 사항 적용:
   - 중복 로직을 서브 워크플로우로 분리
   - 복잡한 Code 노드를 단순화하거나 분리
   - 노드 이름을 역할이 명확한 이름으로 변경
   - 불필요한 노드 제거
4. n8n-mcp로 워크플로우 업데이트

**리팩토링 후에도 기능은 동일하게 유지되어야 한다.**
