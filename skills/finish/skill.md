---
name: n8n-studio:finish
description: This skill should be used when the n8n-studio development cycle reaches the completion phase (8단계) after all verification tests pass, or when the user asks to "작업 마무리해줘", "PR 만들어줘", "결과 정리해줘". Generates 04-result.md, downloads workflows, commits all changes, and creates a GitHub PR.
argument-hint: "[docs 폴더 경로 (선택)]"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep", "Agent", "mcp__n8n-mcp__n8n_get_workflow", "mcp__plugin_github_github__create_pull_request", "mcp__plugin_github_github__create_branch", "mcp__plugin_github_github__push_files"]
---

# n8n-studio:finish

8단계 작업 마무리를 담당한다. 모든 검증이 통과된 후 결과를 정리하고 PR을 생성한다.

## 처리 흐름

workflow-finisher 에이전트를 통해 `summarize-result` 스킬로 진행한다.

### 1. 04-result.md 작성

`docs/yyyyMMdd-기능요약/04-result.md`에 아래 내용을 작성한다:

```markdown
# 작업 결과 요약

## 작업 개요
- 요청 유형: [신규 개발 / 기능 변경 / 버그 수정]
- 대상 워크플로우: [목록]
- 작업 일시: [날짜]

## 변경 사항
- [변경 항목 목록]

## 검증 결과
- 실행한 테스트 시나리오: [N]개
- 통과: [N]개 / 실패: 0개
- RALF 사이클: [N]회

## 주요 결정 사항
- [설계 시 중요한 결정들]
```

### 2. 워크플로우 JSON 다운로드

n8n-mcp로 개발/수정된 워크플로우를 다운로드하여 로컬에 저장한다:
- 저장 경로: `workflow/도메인명/워크플로우명.json` (기존 구조에 맞게 적절한 폴더 선택)
- 워크플로우가 신규인 경우 적절한 서브디렉토리 생성

### 3. Git 작업

```bash
git add docs/yyyyMMdd-기능요약/ workflow/ workflow/test/
git commit -m "feat: [워크플로우명] [변경 요약]"
git push origin feature/작업요약
```

### 4. GitHub PR 생성

**PR 제목:** `feat: [워크플로우명] [변경 내용 요약]`

**PR body 형식:**
```markdown
## Summary
- [변경 사항 bullet point 1-3개]

## Test plan
- [ ] [검증 시나리오 1]
- [ ] [검증 시나리오 2]
- [ ] 통합 테스트 전체 통과 확인

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

**base branch:** 항상 `main`

### 5. 완료 보고

PR URL을 사용자에게 알리며 작업 완료를 보고한다.
