# n8n-studio

A Claude Code plugin for developing n8n workflows through a standardized 8-stage cycle.

> Korean version: [README.md](./README.md)

---

## Overview

n8n-studio is a Claude Code plugin that automates the n8n workflow development process. It handles everything from request analysis, design, TDD-based development, integration testing, to GitHub PR creation — all through a systematic 8-stage cycle.

```
Initiate → Plan → Design → Develop → (Refactor) → Verify → Finish
```

### Key Features

- **8-stage standard cycle**: Consistent process for reliable workflow quality
- **TDD-based Code node development**: Tests pass locally before porting to n8n
- **RALF loop auto-retry**: Up to 3 automatic fix cycles on verification failure
- **Isolated sub-agent architecture**: Each stage runs in a separate context
- **Automatic PR creation**: GitHub PR generated automatically after verification

---

## Prerequisites

- [Claude Code](https://claude.ai/code) installed
- [n8n-mcp MCP server](https://github.com/czlonkowski/n8n-mcp) configured and connected
- Running n8n instance
- GitHub CLI (`gh`) installed (required for automatic PR creation)

---

## Installation

### Step 1: Add marketplace

Run the following command in Claude Code:

```
/plugin add-marketplace icednut/n8n-studio
```

### Step 2: Install plugin

```
/plugin install n8n-studio@n8n-studio
```

### Verify installation

```
/plugin list
```

Installation is complete when `n8n-studio` appears in the list.

---

## Usage

### Basic usage

New workflow development, feature additions, and bug fixes all start the same way:

```
/n8n-studio [your request]
```

Or simply tell Claude:

```
Add a DM feature to the Slack notification workflow
```

```
Create a workflow that saves webhook data to Google Sheets
```

```
There's an error in the order processing workflow. Please fix it.
```

### 8-Stage Development Cycle

| Stage | Agent | Description |
|-------|-------|-------------|
| 1. Classify request | Orchestrator | Classify as new development / feature change / bug fix |
| 2. Gather info | Orchestrator | Confirm workflow IDs, Execution IDs |
| 3. Plan (spec) | workflow-planner | Clarify requirements → write `spec.md`, create branch |
| 4. Design | workflow-designer | Node layout, data flow design → write `design.md` and test files |
| 5. Develop | workflow-developer | TDD Code node development → apply to n8n workflow |
| 6. Refactor | workflow-developer | (Optional) Remove unnecessary complexity |
| 7. Verify | workflow-verifier | Run integration tests, RALF loop (up to 3 cycles) |
| 8. Finish | workflow-finisher | Write result.md, save workflow JSON, git commit, create PR |

### Generated File Structure

```
your-n8n-project/
├── docs/
│   └── 20260410-feature-name/
│       ├── spec.md          # Requirements specification
│       ├── design.md        # Design document
│       ├── test-plan.md     # Test plan
│       └── result.md        # Work result summary
└── workflow/
    ├── test/
    │   ├── unit/            # Code node unit tests (.test.js)
    │   └── acceptance/      # Integration test scenarios (JSON)
    └── [domain]/
        └── workflow-name.json  # n8n workflow JSON backup
```

---

## Included Skills

| Skill | Description |
|-------|-------------|
| `n8n-studio:start` | Main orchestrator for the 8-stage cycle |
| `n8n-studio:analyze-workflow` | Analyze current workflow state |
| `n8n-studio:write-spec` | Write requirements specification |
| `n8n-studio:design` | Run design phase |
| `n8n-studio:write-design` | Write design document |
| `n8n-studio:plan-tests` | Plan test scenarios |
| `n8n-studio:develop` | Run development phase |
| `n8n-studio:develop-code-node` | TDD-based Code node development |
| `n8n-studio:build-workflow` | Create/modify n8n workflows |
| `n8n-studio:refactor` | Refactor workflows |
| `n8n-studio:verify` | Run verification phase |
| `n8n-studio:run-verification` | Execute integration tests |
| `n8n-studio:finish` | Run completion phase |
| `n8n-studio:summarize-result` | Summarize results and create PR |
| `n8n-studio:summarize-workflow` | Analyze workflow node structure and generate JSON summary file |

---

## Included Agents

| Agent | Model | Role |
|-------|-------|------|
| `workflow-planner` | Haiku | Analyze requirements and write spec.md |
| `workflow-designer` | Sonnet | Write design documents and test scenarios |
| `workflow-developer` | Sonnet | TDD development and n8n workflow implementation |
| `workflow-verifier` | Haiku | Run integration tests and evaluate results |
| `workflow-finisher` | Haiku | Summarize results, git commit, create PR |

---

## Recommended Companion Plugin

- **[n8n-mcp-skills](https://github.com/czlonkowski/n8n-skills)**: A collection of expert skills for n8n node configuration, expression syntax, workflow patterns, and more — referenced internally by n8n-studio. Installing alongside n8n-studio is strongly recommended.

```
/plugin add-marketplace czlonkowski/n8n-skills
/plugin install n8n-mcp-skills@n8n-mcp-skills
```

---

## License

MIT License

---

## Contributing

Issues and PRs are welcome at the [GitHub repository](https://github.com/icednut/n8n-studio).
