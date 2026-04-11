---
name: n8n-studio:develop-code-node
description: This skill should be used by n8n-studio agents when they need to develop a Code node using TDD (Test-Driven Development). Follows Method B: write local .js files first, run npm test, then port passing code to n8n Code node.
---

# develop-code-node

단일 책임: TDD 방식으로 n8n Code 노드를 개발하고 `n8n_update_partial_workflow`로 해당 노드만 반영한다.

## 사전 준비: 기존 파일 확인

### 기존 js 파일이 있는 경우

`workflow/[프로젝트명]/js/[워크플로우 이름] - [Code 노드명].js` 파일이 이미 존재하는 경우:

1. **현재 n8n 노드 코드 가져오기**: `mcp__n8n-mcp__n8n_get_workflow`로 워크플로우를 조회하여 해당 Code 노드의 `parameters.jsCode` 추출
2. **js 파일에 반영**: 가져온 코드를 `workflow/[프로젝트명]/js/[워크플로우 이름] - [Code 노드명].js`에 덮어쓰기
3. **기존 Unit Test 확인**: `workflow/[프로젝트명]/test/unit/[워크플로우 이름] - [Code 노드명].test.js` 존재 여부 확인
   - 존재하면: 기존 테스트를 검토하고 개선 후 TDD 진행
   - 없으면: 새로 작성 후 TDD 진행

### 기존 파일이 없는 경우

아래 TDD 순서를 처음부터 진행한다.

---

## TDD 개발 순서

### 1. 테스트 파일 작성 (Red)

`workflow/[프로젝트명]/test/unit/[워크플로우 이름] - [Code 노드명].test.js` 작성:

```javascript
// workflow/[프로젝트명]/test/unit/Task Board - Command Router.test.js
const { processItems } = require('../../js/Task Board - Command Router');

// n8n 전용 변수/함수 Mocking
const mockInput = {
  all: () => [
    { json: { command: '/task', text: 'list' } },
    { json: { command: '/task', text: 'create' } },
  ],
  first: () => ({ json: { command: '/task', text: 'list' } }),
};

// $node, $env, $execution 등 n8n 글로벌 변수 Mock
const mockN8nContext = {
  $node: { name: 'Command Router' },
  $env: { SLACK_TOKEN: 'test-token' },
  $execution: { id: 'exec-001' },
};

describe('Command Router Code Node', () => {
  test('정상 케이스: /task 명령어를 라우팅한다', () => {
    const result = processItems(mockInput, mockN8nContext);
    expect(result).toEqual([
      { json: { route: 'task-list' } },
      { json: { route: 'task-create' } },
    ]);
  });

  test('경계 케이스: 빈 입력 처리', () => {
    const emptyInput = { all: () => [], first: () => null };
    const result = processItems(emptyInput, mockN8nContext);
    expect(result).toEqual([]);
  });

  test('엣지 케이스: 알 수 없는 명령어', () => {
    const unknownInput = {
      all: () => [{ json: { command: '/unknown' } }],
      first: () => ({ json: { command: '/unknown' } }),
    };
    const result = processItems(unknownInput, mockN8nContext);
    expect(result[0].json.route).toBe('unknown');
  });
});
```

**n8n 변수 Mocking 규칙:**

| n8n 변수 | Mock 방법 |
|---------|---------|
| `$input` | `{ all: () => [...], first: () => ({...}) }` |
| `$json` | 각 테스트에서 직접 데이터 구성 |
| `$node['이름'].json` | `mockN8nContext.$node` 또는 별도 객체 |
| `$env` | `mockN8nContext.$env = { KEY: 'value' }` |
| `$execution` | `mockN8nContext.$execution = { id: '...' }` |

### 2. 테스트 실행 → 실패 확인 (Red)

```bash
npx jest "workflow/[프로젝트명]/test/unit/[워크플로우 이름] - [Code 노드명].test.js" --no-coverage
```

RED 상태(테스트 실패) 확인 후 다음 단계 진행.

### 3. 구현 파일 작성 (Green)

`workflow/[프로젝트명]/js/[워크플로우 이름] - [Code 노드명].js` 작성:

```javascript
// workflow/[프로젝트명]/js/Task Board - Command Router.js
// n8n Code 노드에서 사용할 로직 (테스트 가능한 함수 형태)

/**
 * @param {object} input - n8n $input 객체 (또는 Mock)
 * @param {object} ctx - n8n 글로벌 변수 컨텍스트 (또는 Mock)
 */
function processItems(input, ctx = {}) {
  const items = input.all();
  return items.map(item => {
    const command = item.json.command || '';
    const text = item.json.text || '';

    let route = 'unknown';
    if (command === '/task' && text === 'list') route = 'task-list';
    else if (command === '/task' && text === 'create') route = 'task-create';

    return { json: { route } };
  });
}

module.exports = { processItems };
```

**구현 규칙:**
- 함수는 `input`과 선택적 `ctx`를 인자로 받아 테스트 가능하게 작성
- `module.exports`로 내보내어 테스트에서 `require`로 사용
- `require()` 가능 모듈: `simple-git`, `yaml`, `glob`

### 4. 전체 테스트 실행 → 통과 확인 (Green)

```bash
npx jest "workflow/[프로젝트명]/test/unit/[워크플로우 이름] - [Code 노드명].test.js" --no-coverage
```

실패하는 테스트가 있으면 구현 보완 후 재실행 — 모두 통과할 때까지 반복.

### 5. n8n 워크플로우에 반영

통과 확인 후, 구현 파일의 로직을 n8n Code 노드 형식으로 변환하여 `n8n_update_partial_workflow`로 해당 노드만 업데이트한다.

**n8n Code 노드 형식으로 변환 (이식 코드 예시):**

```javascript
// n8n Code 노드에 들어갈 실제 코드
// module.exports, function 선언 제거 → 직접 실행 코드로 변환
// $input, $json 등 n8n 글로벌 변수를 직접 사용

const items = $input.all();
const results = [];

for (const item of items) {
  const command = item.json.command || '';
  const text = item.json.text || '';

  let route = 'unknown';
  if (command === '/task' && text === 'list') route = 'task-list';
  else if (command === '/task' && text === 'create') route = 'task-create';

  results.push({ json: { route } });
}

return results;
```

**`n8n_update_partial_workflow` 호출 예시:**

```json
{
  "workflowId": "[workflow_id]",
  "nodes": [
    {
      "id": "[node_id]",
      "parameters": {
        "jsCode": "[이식된 코드 문자열]"
      }
    }
  ]
}
```

- `workflowId`: 대상 워크플로우 ID
- `nodes[].id`: 업데이트할 Code 노드의 ID (summary 파일 또는 `n8n_get_workflow`에서 확인)
- `nodes[].parameters.jsCode`: 이식된 코드 문자열

**이식 주의 사항:**
- `module.exports`, `require('./파일명')` 제거
- 로컬 함수 호출 대신 인라인 로직으로 변환
- `$input`, `$json`, `$node` 등 n8n 글로벌 변수 직접 사용
- n8n 표현식 문법은 `n8n-mcp-skills:n8n-expression-syntax` 참조
