# agentic-workflow

5-stage development workflow plugin for Claude Code.

```
SCOPE → DESIGN → BUILD → HARDEN → PR
```

For developers whose endpoint is a pull request, not production deployment.

## Quick Start

```
/workflow <task description>
```

Runs all 5 stages sequentially with approval gates between each. Individual stages (`/scope`, `/design`, etc.) are also callable standalone.

## Stages

| Stage | Command | Purpose |
|-------|---------|---------|
| | `/workflow` | **Orchestrator** — runs all stages with approval gates |
| **SCOPE** | `/scope` | Define goal, non-goals, and delivery slices |
| **DESIGN** | `/design` | Schema-first design with data flow and boundaries |
| **BUILD** | `/build` | Implement one slice at a time with per-slice verification |
| **HARDEN** | `/harden` | Integration tests, LLM safety audit, dead code cleanup |
| **PR** | `/pr` | Self-review, write PR body, submit |

## Install

```
/plugin marketplace add yoonjong12/agentic-workflow
/plugin install agentic-workflow@agentic-workflow
```

## References

- `references/structured-output-checklist.md` — Pydantic + LLM safety patterns
- `references/integration-test-checklist.md` — Multi-component test patterns
- `references/pr-review-checklist.md` — Self-review before submit

## Design Principles

- **Process over prose** — Steps to follow, not docs to read
- **Anti-rationalization** — Each stage includes rebuttals to common skip-excuses
- **Verification is non-negotiable** — Evidence required at every gate
- **Entry/exit criteria** — Clear conditions for stage transitions
