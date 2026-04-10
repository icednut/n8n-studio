# n8n-studio

Claude Code에서 n8n 워크플로우를 표준화된 8단계 사이클로 개발하기 위한 플러그인입니다.

> 영문 버전: [README.en.md](./README.en.md)

---

## 개요

n8n-studio는 n8n 워크플로우 개발 과정을 자동화하는 Claude Code 플러그인입니다. 요청 분석부터 설계, TDD 기반 개발, 통합 테스트, GitHub PR 생성까지 전 과정을 8단계 사이클로 체계적으로 진행합니다.

```
발단 → 계획 → 설계 → 개발 → (리팩토링) → 검증 → 마무리
```

### 주요 특징

- **8단계 표준 사이클**: 일관된 프로세스로 워크플로우 품질 확보
- **TDD 기반 Code 노드 개발**: 로컬에서 테스트 통과 후 n8n에 이식
- **RALF 루프 자동 재시도**: 검증 실패 시 최대 3회 자동 수정
- **독립 서브에이전트 아키텍처**: 각 단계가 격리된 컨텍스트로 실행
- **자동 PR 생성**: 검증 완료 후 GitHub PR까지 자동 생성

---

## 사전 요구사항

- [Claude Code](https://claude.ai/code) 설치
- [n8n-mcp MCP 서버](https://github.com/czlonkowski/n8n-mcp) 설정 및 연결
- n8n 인스턴스 실행 중
- GitHub CLI (`gh`) 설치 (PR 자동 생성 사용 시)

---

## 설치

### 1단계: 마켓플레이스 등록

Claude Code에서 아래 명령어를 실행합니다:

```
/plugin add-marketplace icednut/n8n-studio
```

### 2단계: 플러그인 설치

```
/plugin install n8n-studio@n8n-studio
```

### 설치 확인

```
/plugin list
```

목록에 `n8n-studio`가 표시되면 설치 완료입니다.

---

## 사용법

### 기본 사용법

새 워크플로우 개발, 기능 추가, 버그 수정 모두 동일한 방법으로 시작합니다:

```
/n8n-studio [요청 내용]
```

또는 Claude에게 직접 말하세요:

```
슬랙 알림 워크플로우에 DM 기능을 추가해줘
```

```
Webhook으로 받은 데이터를 Google Sheets에 저장하는 워크플로우 만들어줘
```

```
주문 처리 워크플로우에서 오류가 발생하고 있어. 고쳐줘.
```

### 8단계 개발 사이클

| 단계 | 담당 에이전트 | 내용 |
|------|--------------|------|
| 1. 요청 분류 | 오케스트레이터 | 신규 개발 / 기능 변경 / 버그 수정 분류 |
| 2. 정보 수집 | 오케스트레이터 | 워크플로우 ID, Execution ID 확인 |
| 3. 계획 (spec) | workflow-planner | 요구사항 구체화 → `spec.md` 작성, 브랜치 생성 |
| 4. 설계 (design) | workflow-designer | 노드 구성, 데이터 흐름 설계 → `design.md`, 테스트 파일 작성 |
| 5. 개발 | workflow-developer | TDD로 Code 노드 개발 → n8n 워크플로우 반영 |
| 6. 리팩토링 | workflow-developer | (선택) 불필요한 복잡도 제거 |
| 7. 검증 | workflow-verifier | 통합 테스트 실행, RALF 루프 (최대 3회) |
| 8. 마무리 | workflow-finisher | result.md 작성, 워크플로우 JSON 저장, git commit, PR 생성 |

### 생성되는 파일 구조

```
your-n8n-project/
├── docs/
│   └── 20260410-feature-name/
│       ├── spec.md          # 요구사항 명세
│       ├── design.md        # 설계 문서
│       ├── test-plan.md     # 테스트 계획
│       └── result.md        # 작업 결과 요약
└── workflow/
    ├── test/
    │   ├── unit/            # Code 노드 단위 테스트 (.test.js)
    │   └── acceptance/      # 통합 테스트 시나리오 (JSON)
    └── [domain]/
        └── workflow-name.json  # n8n 워크플로우 JSON 백업
```

---

## 포함된 스킬 목록

| 스킬 | 설명 |
|------|------|
| `n8n-studio:start` | 8단계 사이클 메인 오케스트레이터 |
| `n8n-studio:analyze-workflow` | 워크플로우 현재 상태 분석 |
| `n8n-studio:write-spec` | 요구사항 명세서 작성 |
| `n8n-studio:design` | 설계 단계 진행 |
| `n8n-studio:write-design` | 설계 문서 작성 |
| `n8n-studio:plan-tests` | 테스트 시나리오 계획 |
| `n8n-studio:develop` | 개발 단계 진행 |
| `n8n-studio:develop-code-node` | TDD 방식 Code 노드 개발 |
| `n8n-studio:build-workflow` | n8n 워크플로우 생성/수정 |
| `n8n-studio:refactor` | 워크플로우 리팩토링 |
| `n8n-studio:verify` | 검증 단계 진행 |
| `n8n-studio:run-verification` | 통합 테스트 실행 |
| `n8n-studio:finish` | 마무리 단계 진행 |
| `n8n-studio:summarize-result` | 결과 정리 및 PR 생성 |

---

## 포함된 에이전트 목록

| 에이전트 | 모델 | 역할 |
|---------|------|------|
| `workflow-planner` | Haiku | 요구사항 분석 및 spec.md 작성 |
| `workflow-designer` | Sonnet | 설계 문서 및 테스트 시나리오 작성 |
| `workflow-developer` | Sonnet | TDD 개발 및 n8n 워크플로우 구현 |
| `workflow-verifier` | Haiku | 통합 테스트 실행 및 결과 판단 |
| `workflow-finisher` | Haiku | 결과 정리, git 커밋, PR 생성 |

---

## 함께 사용하면 좋은 플러그인

- **[n8n-mcp-skills](https://github.com/czlonkowski/n8n-skills)**: n8n 노드 설정, 표현식 문법, 워크플로우 패턴 등 n8n-studio 내부에서 참조하는 전문 스킬 모음. n8n-studio와 함께 설치를 권장합니다.

```
/plugin add-marketplace czlonkowski/n8n-skills
/plugin install n8n-mcp-skills@n8n-mcp-skills
```

---

## 라이선스

MIT License

---

## 기여

이슈 및 PR은 [GitHub 저장소](https://github.com/icednut/n8n-studio)에서 환영합니다.
