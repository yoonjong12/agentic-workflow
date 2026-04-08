---
name: workflow
description: "Orchestrate the full development workflow from task to PR. Trigger on: 'workflow', 'start workflow', 'new task', 'start working', '워크플로', '작업 시작', 'kick off'"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, Skill
argument-hint: "<task description or Jira ticket>"
---

# WORKFLOW — Orchestrator

Run the full development pipeline: **SCOPE → DESIGN → BUILD → HARDEN → PR**.

One command starts it. Each stage runs, gets user approval, then the next begins.

## Process

### 1. Initialize

Create `.workflow-state.md` in the working directory to track progress:

```markdown
# Workflow: <task one-liner>
Started: <date>

## Progress
- [ ] SCOPE
- [ ] DESIGN
- [ ] BUILD
- [ ] HARDEN
- [ ] PR
```

### 2. Run stages sequentially

For each stage in order:

**a) Announce the stage:**
```
━━━ STAGE: SCOPE (1/5) ━━━
```

**b) Execute the stage skill** by loading and following its full process:
- SCOPE → follow `skills/scope/SKILL.md` process
- DESIGN → follow `skills/design/SKILL.md` process
- BUILD → follow `skills/build/SKILL.md` process
- HARDEN → follow `skills/harden/SKILL.md` process
- PR → follow `skills/pr/SKILL.md` process

**c) Verify exit criteria** from the stage's SKILL.md. List each criterion with pass/fail.

**d) Ask for approval before advancing:**
```
SCOPE exit criteria:
  ✓ scope.md exists with goal, non-goals, slices
  ✓ At least 1 non-goal
  ✓ Every slice has verification method

→ Proceed to DESIGN? (y / revise / stop)
```

Three responses:
- **y** → advance to next stage
- **revise** → user gives feedback, re-run current stage with adjustments
- **stop** → pause workflow, save state to `.workflow-state.md`

**e) Update `.workflow-state.md`** — mark stage `[x]` on approval.

### 3. Resume

If a workflow was stopped mid-way, check for `.workflow-state.md` and resume from the last incomplete stage. Do not re-run completed stages.

### 4. Complete

After PR stage:
```
━━━ WORKFLOW COMPLETE ━━━
All 5 stages passed. PR submitted.
```

Clean up `.workflow-state.md` or leave for reference.

## Rules

- **Never skip a stage.** If the user says "just build it," explain why SCOPE and DESIGN exist and offer a lightweight version — not a skip.
- **Never auto-approve.** Every stage gate requires explicit user approval.
- **Individual stages still work.** `/scope`, `/design`, etc. remain independently callable. The orchestrator is a convenience, not a requirement.
- **Respect stops.** When the user says stop, save state immediately. Do not ask "are you sure?"
