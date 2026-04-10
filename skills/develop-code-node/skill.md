---
name: n8n-studio:develop-code-node
description: This skill should be used by n8n-studio agents when they need to develop a Code node using TDD (Test-Driven Development). Follows Method B: write local .js files first, run npm test, then port passing code to n8n Code node.
---

# develop-code-node

단일 책임: TDD 방식(방식 B)으로 n8n Code 노드를 개발한다.

## TDD 개발 순서

### 1. 테스트 파일 작성 (Red)

설계 문서의 Code 노드 스펙을 보고 `workflow/test/unit/[기능명].test.js` 작성:

```javascript
// workflow/test/unit/[기능명].test.js
const { [함수명] } = require('./[기능명]');

describe('[Code 노드 이름]', () => {
  test('정상 케이스: [설명]', () => {
    const mockInput = {
      all: () => [{ json: { /* 입력 데이터 */ } }],
      first: () => ({ json: { /* 첫 번째 입력 */ } }),
    };
    const result = [함수명](mockInput);
    expect(result).toEqual([{ json: { /* 기대 출력 */ } }]);
  });

  test('경계 케이스: [설명]', () => {
    // ...
  });
});
```

### 2. 테스트 실행 → 실패 확인

```bash
npm test -- workflow/test/unit/[기능명].test.js
```

RED 상태 (테스트 실패) 확인 후 다음 단계 진행.

### 3. 구현 파일 작성 (Green)

`workflow/test/unit/[기능명].js` 작성:

```javascript
// n8n Code 노드에서 사용할 로직
function [함수명](input) {
  const items = input.all();
  // 구현 로직
  return items.map(item => ({
    json: {
      // 변환된 출력
    }
  }));
}

module.exports = { [함수명] };
```

**n8n Code 노드 변수 규칙 (`n8n-mcp-skills:n8n-code-javascript` 참조):**
- `$input.all()` — 모든 입력 항목
- `$input.first()` — 첫 번째 입력 항목
- `$json` — 현재 항목의 JSON 데이터
- `$node['노드이름'].json` — 특정 노드 출력 접근

### 4. 테스트 실행 → 통과 확인

```bash
npm test -- workflow/test/unit/[기능명].test.js
```

GREEN 상태 (테스트 통과) 확인.

### 5. n8n Code 노드에 이식

로컬 구현 파일의 핵심 로직을 n8n Code 노드 형식으로 변환:

```javascript
// n8n Code 노드 (Run Once for All Items 모드)
const results = [];
for (const item of $input.all()) {
  // [기능명].js의 로직을 여기에 직접 이식
  results.push({ json: { /* 출력 */ } });
}
return results;
```

**이식 시 주의 사항:**
- `require()` 사용 가능 모듈: `simple-git`, `yaml`, `glob`
- `module.exports`나 함수 선언 제거, 직접 실행 코드로 변환
- n8n 표현식 문법 확인: `n8n-mcp-skills:n8n-expression-syntax` 참조
