# agentic-workflow

5-stage development workflow plugin for Claude Code.

```
SCOPE → DESIGN → BUILD → HARDEN → PR
```

For developers whose endpoint is a pull request, not production deployment.

## Stages

| Stage | Command | Purpose |
|-------|---------|---------|
| **SCOPE** | `/scope` | Define goal, non-goals, and delivery slices |
| **DESIGN** | `/design` | Schema-first design with data flow and boundaries |
| **BUILD** | `/build` | Implement one slice at a time with per-slice verification |
| **HARDEN** | `/harden` | Integration tests, LLM safety audit, dead code cleanup |
| **PR** | `/pr` | Self-review, write PR body, submit |

## Install

```bash
# Local plugin
claude --plugin-dir /path/to/agentic-workflow

# Or add to project
cp -r agentic-workflow/.claude-plugin /your/project/
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
