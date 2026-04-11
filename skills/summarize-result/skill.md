---
name: n8n-studio:summarize-result
description: This skill should be used by n8n-studio agents when they need to write the final 04-result.md, download workflow JSONs to the local workflow/ directory, commit all changes to git, and create a GitHub PR.
---

# summarize-result

단일 책임: 작업 결과를 정리하고 git commit 및 GitHub PR 생성으로 마무리한다.

## 수행 작업

### 1. 04-result.md 작성

`docs/[프로젝트명]/yyyyMMdd-기능요약/04-result.md`:

```markdown
# 작업 결과 요약

## 작업 개요
- 요청 유형: [신규 개발 / 기능 변경 / 버그 수정]
- 대상 워크플로우: [이름 및 ID]
- 작업 일시: [yyyyMMdd]

## 변경 사항
- [변경 항목 1]
- [변경 항목 2]

## 검증 결과
- 실행한 테스트 시나리오: N개
- 통과: N개 / 실패: 0개
- RALF 사이클: N회

## 주요 결정 사항
- [설계/개발 중 중요한 결정들]
```

### 2. 워크플로우 JSON 저장

개발/수정된 워크플로우를 n8n-mcp로 다운로드하여 로컬에 저장:
- n8n-mcp의 `n8n_get_workflow`로 워크플로우 JSON 가져오기
- 저장 경로: `workflow/[프로젝트명]/n8n/[워크플로우명].json`
  - 기존에 파일이 있으면 덮어쓰기
  - `n8n/` 폴더가 없으면 생성

> **주의**: 폴더명은 반드시 `workflow/`(단수, s 없음)를 사용한다. `workflows/`(복수형) 폴더를 생성하거나 사용하지 않는다.

### 3. Git 작업

```bash
git add docs/[프로젝트명]/yyyyMMdd-기능요약/ workflow/[프로젝트명]/n8n/ workflow/[프로젝트명]/test/
git commit -m "feat: [워크플로우명] [변경 내용 한 줄 요약]

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
git push origin feature/작업요약
```

### 4. GitHub PR 생성

`mcp__plugin_github_github__create_pull_request` 사용:

- **제목**: `feat: [워크플로우명] [변경 내용 요약]`
- **base**: `main`
- **head**: `feature/작업요약`

**body 형식:**
```markdown
## Summary
- [변경 사항 1]
- [변경 사항 2]

## Test plan
- [ ] [검증 시나리오 1]
- [ ] [검증 시나리오 2]
- [ ] 통합 테스트 전체 통과 확인 (N개 시나리오)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

### 5. 완료 보고

생성된 PR URL을 포함하여 작업 완료를 사용자에게 보고한다.
