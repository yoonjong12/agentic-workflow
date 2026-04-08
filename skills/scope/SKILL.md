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

## Process

### Step 1: One-sentence goal

Write the goal as a single sentence. If it needs two sentences, the scope is too big — split first.

```
Goal: Add curation feedback loop so PCR accumulates evidence from MEGA reflections.
```

**Checkpoint**: Can someone unfamiliar with the project understand what "done" looks like from this sentence alone?

### Step 2: Non-goals (minimum 1)

List things that are adjacent but explicitly OUT of scope. Non-goals prevent scope creep more effectively than goals do.

```
Non-goals:
- NOT changing the existing PCR search API contract
- NOT adding a UI for feedback visualization
- NOT optimizing feedback storage performance
```

**Checkpoint**: For each non-goal, ask "would a reasonable person assume this IS in scope?" If yes, it belongs here.

### Step 3: Slice into deliverable units

Break the work into 3-7 slices. Each slice:
- Is independently verifiable (test, manual check, or observable output)
- Takes 3-5 commits maximum
- Can be reverted without breaking other slices

```
Slices:
1. CurationFeedback Pydantic model + validation → verify: model_validate round-trip
2. POST /feedback endpoint → verify: curl returns 201
3. Reflection pattern extraction → verify: unit test with known input
4. Curation reference injection → verify: integration test shows injected context
```

**Checkpoint**: If any slice has no verification method written next to it, it's not a slice — it's a wish.

### Step 4: Write scope.md

Create `scope.md` in the working directory (or `.claude/scope.md` for cross-session persistence):

```markdown
# Scope: <goal one-liner>

## Goal
<one sentence>

## Non-goals
- <item>

## Slices
1. <slice> → verify: <how>
2. ...

## Open questions
- <anything unresolved>
```

## Rationalizations — why you'll want to skip this

| Excuse | Counter |
|--------|---------|
| "It's small, I don't need a scope" | feat/curation-feedback started "small" and hit 52 commits. The cost of scope.md is 5 minutes. The cost of no scope is an unreverable branch. |
| "I'll figure out non-goals as I go" | By the time you realize something is out of scope mid-implementation, you've already built half of it. |
| "Slicing takes longer than just coding" | Slicing takes 10 minutes. Debugging a 15-commit slice with no rollback point takes hours. |

## Red Flags

- Goal sentence contains "and" → likely two tasks, split them
- Zero non-goals → you haven't thought about boundaries
- A slice has no verification method → it will be "done" when it "feels done"
- More than 7 slices → scope is too large, split into multiple tasks
- A slice needs more than 5 commits → break it down further

## Exit Criteria

All must be true:
- [ ] `scope.md` exists with goal, non-goals, and slices
- [ ] At least 1 non-goal defined
- [ ] Every slice has a verification method
- [ ] No slice exceeds 5 estimated commits
- [ ] Open questions section exists (even if empty)

## Next → /design
