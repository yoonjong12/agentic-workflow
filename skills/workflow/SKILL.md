---
name: workflow
description: "Orchestrate the full development workflow from task to PR. Trigger on: 'workflow', 'start workflow', 'new task', 'start working', '워크플로', '작업 시작', 'kick off'"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, Skill
argument-hint: "<task description or Jira ticket>"
---

# WORKFLOW — Orchestrator

Run the full development pipeline: **SCOPE → DESIGN → BUILD → HARDEN → PR**.

One command starts it. Each stage runs, gets user approval, then the next begins.

Each workflow lives in its own **session directory** under `.claude/workflows/<session-id>/` so multiple tasks can run in parallel without overwriting each other.

## Process

### 0. Session initialization

Before running any stage, establish the session.

**a) Propose a session ID** from the task argument:
- If the task mentions a Jira/issue key (e.g., `WAO-389`, `PROJ-123`), use it verbatim.
- Otherwise, generate a short kebab-case slug from the task (3–5 words, e.g., `curation-feedback-rebase`).
- If `.claude/workflows/<id>/` already exists, append `-02`, `-03`, etc.

Show the proposed ID to the user:
```
Proposed session ID: WAO-389-rebase
  → Directory: .claude/workflows/WAO-389-rebase/
Use this? (y / <custom-id>)
```

**b) Create the session directory**:
```bash
mkdir -p .claude/workflows/<session-id>
```

**c) Record the active session**:
```bash
echo "<session-id>" > .claude/workflows/.active
```

All subsequent skills (`/scope`, `/design`, `/build`, `/harden`, `/pr`) read `.claude/workflows/.active` to resolve their working directory. Do not write to hardcoded paths like `./scope.md` or `.claude/scope.md`.

**d) Initialize `<session-dir>/state.md`**:

```markdown
# Workflow: <task one-liner>
Session: <session-id>
Started: <date>

## Progress
- [ ] SCOPE
- [ ] DESIGN
- [ ] BUILD
- [ ] HARDEN
- [ ] PR
```

### 1. Resume check

If `.claude/workflows/.active` already points to an existing session with `state.md` progress, ask:
```
Active session: <id> — progress: SCOPE [x] DESIGN [ ] ...
Resume this session, or start a new one? (resume / new)
```
Do not re-run completed stages on resume.

### 2. Run stages sequentially

For each stage in order:

**a) Announce the stage:**
```
━━━ STAGE: SCOPE (1/5) — session <id> ━━━
```

**b) Execute the stage skill** by loading and following its full process:
- SCOPE → follow `skills/scope/SKILL.md` process (writes `<session-dir>/scope.md`)
- DESIGN → follow `skills/design/SKILL.md` process (writes `<session-dir>/design.md`)
- BUILD → follow `skills/build/SKILL.md` process
- HARDEN → follow `skills/harden/SKILL.md` process
- PR → follow `skills/pr/SKILL.md` process

**c) Verify exit criteria** from the stage's SKILL.md. List each criterion with pass/fail.

**d) Ask for approval before advancing:**
```
SCOPE exit criteria:
  ✓ <session-dir>/scope.md exists with goal, non-goals, slices
  ✓ At least 1 non-goal
  ✓ Every slice has verification method

→ Proceed to DESIGN? (y / revise / stop)
```

Three responses:
- **y** → advance to next stage
- **revise** → user gives feedback, re-run current stage with adjustments
- **stop** → pause workflow, save state to `<session-dir>/state.md` and exit

**e) Update `<session-dir>/state.md`** — mark stage `[x]` on approval.

### 3. Resume

If a workflow was stopped mid-way, `.claude/workflows/.active` still points to the paused session. Read `<session-dir>/state.md` and resume from the last incomplete stage. Do not re-run completed stages.

### 4. Complete

After PR stage:
```
━━━ WORKFLOW COMPLETE — session <id> ━━━
All 5 stages passed. PR submitted.
```

Mark `state.md` fully complete. Leave the session directory for reference; do not delete. Clear `.claude/workflows/.active` (or leave pointing to this session — harmless until a new workflow starts).

## Rules

- **Every workflow gets its own session directory.** Never write scope/design artifacts outside `<session-dir>`.
- **Never skip a stage.** If the user says "just build it," explain why SCOPE and DESIGN exist and offer a lightweight version — not a skip.
- **Never auto-approve.** Every stage gate requires explicit user approval.
- **Individual stages still work.** `/scope`, `/design`, etc. remain independently callable — they resolve the session from `.claude/workflows/.active` and prompt if none is active.
- **Respect stops.** When the user says stop, save state immediately. Do not ask "are you sure?"
