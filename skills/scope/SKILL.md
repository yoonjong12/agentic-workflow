---
name: scope
description: "Define what to do and what NOT to do before any implementation. Trigger on: 'scope', 'scope this', 'what should I build', 'define scope', 'non-goals', 'slice this work', 'break this down', 'scope 잡아', '스코프'"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent
argument-hint: "<task description or Jira ticket>"
---

# SCOPE — What to do, what NOT to do

## Overview

Every runaway branch started as "a small task." SCOPE forces a written contract with yourself: one goal sentence, explicit non-goals, and a sliced delivery plan with per-slice verification. No code until this exists.

## Entry Criteria

- A task exists (Jira ticket, verbal request, or your own idea)
- No implementation has started yet for this task

## Session Resolution

This skill writes `scope.md` under a session directory, never to hardcoded `./scope.md` or `.claude/scope.md`.

Resolve `<session-dir>` before Step 0:

1. Read `.claude/workflows/.active`. If it exists and contains an ID, set `<session-dir> = .claude/workflows/<id>/`.
2. If `.active` is missing:
   - **Invoked from `/workflow`**: workflow already created the session — use the ID it propagated.
   - **Invoked standalone** (`/scope` directly): list existing sessions (`ls .claude/workflows/`), then ask:
     ```
     No active session. Options:
       [1] Create new session (propose ID from task)
       [2] Attach to existing: <list>
     ```
     On "new", propose a kebab-case ID (Jira key if task mentions one, else slug from goal), create `.claude/workflows/<id>/`, write `.claude/workflows/.active`.

Use `<session-dir>/scope.md` wherever this document says "scope.md".

## Process

### Step 0: Acceptance — the user's desired end state

Before anything else, capture what the user wants to EXPERIENCE when this is done. Copy their words verbatim or ask them to state it.

```
Acceptance: 메가오프라인에서 /통합테스트 실행 → 전체 루프 통과 → 큐레이션/피드백 동작 보고 받기
```

This is NOT your implementation goal. This is what the user will do to verify you succeeded. You cannot rewrite, narrow, or "translate" this into an implementation task.

**Checkpoint**: Read your Acceptance and your Goal (Step 1) side by side. If the Goal is an implementation task ("fix X", "add Y", "refactor Z") but the Acceptance is an outcome the user experiences ("run X and see Y"), your Goal has drifted. Rewrite the Goal to serve the Acceptance, not replace it.

### Step 1: One-sentence goal

Write the goal as a single sentence. This must be a means to achieve the Acceptance, not a substitute for it.

```
Goal: Add user notification preferences so the API respects per-user channel settings.
```

**Checkpoint**: Can someone unfamiliar with the project understand what "done" looks like from this sentence alone?

### Step 2: Non-goals (minimum 1)

List things that are adjacent but explicitly OUT of scope. Non-goals prevent scope creep more effectively than goals do.

```
Non-goals:
- NOT changing the existing notification delivery pipeline
- NOT adding a UI for preference management
- NOT optimizing notification throughput
```

**Checkpoint**: For each non-goal, ask "would a reasonable person assume this IS in scope?" If yes, it belongs here.

### Step 3: Slice into deliverable units

Break the work into 3-7 slices. Each slice:
- Is independently verifiable (test, manual check, or observable output)
- Takes 3-5 commits maximum
- Can be reverted without breaking other slices

**The last slice must always be Acceptance verification.** This is where you (or the user) actually perform the Acceptance scenario and confirm it works. If some steps require user participation (starting servers, running commands in another terminal), state that explicitly — do not silently drop them.

```
Slices:
1. NotificationPreference Pydantic model + validation → verify: model_validate round-trip
2. GET/PUT /preferences endpoint → verify: curl returns 200 with correct schema
3. Preference-aware delivery logic → verify: unit test with channel filtering
4. Acceptance: send notification with user preferences → verify: user receives on correct channel only
```

**Checkpoint**: If any slice has no verification method written next to it, it's not a slice — it's a wish.

### Step 4: Falsifiability test — one executable check for the Acceptance

Beyond per-slice verification, write a SINGLE executable test that encodes the Acceptance. It must:

- **Fail today** (before any implementation). Run it now, confirm red.
- **Pass only when the full Acceptance is met** — not partial progress, not "most slices done."
- **Be cheap to run** (seconds, not minutes) so /build can execute it after every slice.
- **Live at a stable path** — typically `<session-dir>/falsify.sh` or a dedicated test file.

```bash
# Example: <session-dir>/falsify.sh
#!/usr/bin/env bash
set -e
# The Acceptance: "user receives notification only on their preferred channel"
curl -sf http://localhost:8000/preferences/user123 | jq -e '.channels | length > 0'
pytest tests/test_preference_delivery.py::test_channel_filter_end_to_end -x
```

**Why this is separate from slice verification**: Slice-level checks confirm "I coded slice N correctly." The falsifiability test confirms "the whole story produced the user-visible outcome." Slices can all be green while the test stays red — that's a design gap, caught early.

**Checkpoint**: Run the falsifiability test NOW. It must fail. If it passes, either the feature already exists (task is done, stop) or the test is wrong (too weak — tighten it).

### Step 5: Write scope.md

Create `<session-dir>/scope.md` (resolved in Session Resolution above):

```markdown
# Scope: <goal one-liner>

## Acceptance
> <user's verbatim desired end state — what they will do/see when this is done>

## Goal
<one sentence — a means to achieve Acceptance, not a substitute>

## Non-goals
- <item>

## Slices
1. <slice> → verify: <how>
2. ...
N. Acceptance verification → verify: <perform the Acceptance scenario>

## Falsifiability Test
- Path: `<session-dir>/falsify.sh` (or test file)
- Runs in: <seconds>
- Baseline (before implementation): RED (fails)
- Target (on completion): GREEN

## Open questions
- <anything unresolved>
```

## Rationalizations — why you'll want to skip this

| Excuse | Counter |
|--------|---------|
| "It's small, I don't need a scope" | Every 50-commit branch started as "a quick feature." The cost of scope.md is 5 minutes. The cost of no scope is an unrevertable branch. |
| "I'll figure out non-goals as I go" | By the time you realize something is out of scope mid-implementation, you've already built half of it. |
| "Slicing takes longer than just coding" | Slicing takes 10 minutes. Debugging a 15-commit slice with no rollback point takes hours. |

## Red Flags

- **Goal describes what YOU will code, not what the USER will experience** → you substituted Acceptance with an implementation task. Rewrite.
- **Last slice is a unit test, not the Acceptance scenario** → you optimized for what you can verify alone, dropping the user's actual goal.
- Goal sentence contains "and" → likely two tasks, split them
- Zero non-goals → you haven't thought about boundaries
- A slice has no verification method → it will be "done" when it "feels done"
- More than 7 slices → scope is too large, split into multiple tasks
- A slice needs more than 5 commits → break it down further

## Exit Criteria

All must be true:
- [ ] `<session-dir>/scope.md` exists with Acceptance, goal, non-goals, and slices
- [ ] Acceptance is the user's verbatim desired end state
- [ ] Goal serves the Acceptance (not a substitution of it)
- [ ] Last slice is Acceptance verification
- [ ] At least 1 non-goal defined
- [ ] Every slice has a verification method
- [ ] No slice exceeds 5 estimated commits
- [ ] Open questions section exists (even if empty)
- [ ] Falsifiability test exists at a stable path and FAILS today (baseline red confirmed)

## Next → /design
