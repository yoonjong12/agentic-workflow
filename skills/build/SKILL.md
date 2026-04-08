---
name: build
description: "Implement one slice at a time with verification between each. Trigger on: 'build', 'implement', 'start coding', 'build this', 'implement slice', '구현', '빌드', 'start building'"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent
argument-hint: "<slice number or description>"
---

# BUILD — One slice, one verification

## Overview

BUILD executes one slice from scope.md at a time. Each slice is implemented, verified, and committed before the next begins. No batching, no "I'll test later."

## Entry Criteria

- `scope.md` exists with slices and verification criteria
- `design.md` exists with schemas and boundaries (or design is trivially simple)
- Current slice is identified

## Process

### Step 1: Select the slice

Read scope.md. Pick the next unfinished slice. State it explicitly:

```
Building slice 2: GET/PUT /preferences endpoint → verify: curl returns 200 with correct schema
```

If the slice depends on a previous slice that isn't verified, stop. Go back and verify that first.

**Checkpoint**: The slice and its verification method are stated before any code is written.

### Step 2: Implement within boundaries

Write code for THIS slice only. Rules:

- **Follow design.md boundaries** — if the design says `channel_router.py` owns the transformation, put it there
- **Use the schemas from design.md** — don't invent new models on the fly
- **Touch only what this slice requires** — adjacent code improvements go in a separate commit or not at all
- **No speculative code** — don't build "hooks" for the next slice

Commit cadence: 1-3 commits per slice. Each commit should compile/run.

**Checkpoint**: Every changed line traces back to the current slice description.

### Step 3: Verify the slice

Run the verification method defined in scope.md. This is non-negotiable.

```bash
# slice 1: model round-trip
python -c "from models import UserPreference; UserPreference.model_validate({...})"

# slice 2: endpoint responds
curl -s localhost:8000/preferences/user123 | python -m json.tool

# slice 3: unit test
pytest tests/test_channel_routing.py -v

# slice 4: integration test
pytest tests/integration/test_notification_flow.py -v
```

If verification fails: fix it in this slice. Do not move on.

**Checkpoint**: Verification output is visible (test pass, HTTP 201, no error). "It looks right" is not verification.

### Step 4: Commit and mark slice complete

```bash
git add <specific files> && git commit -m "feat: <what this slice adds>"
```

Update scope.md: mark the slice as done.

```markdown
## Slices
1. [x] UserPreference model → verified: model_validate round-trip passes
2. [ ] GET/PUT /preferences endpoint → verify: curl returns 200   ← NEXT
```

### Step 5: Repeat or exit

If more slices remain → go to Step 1 with next slice.
If all slices done → proceed to /harden.

## Rationalizations

| Excuse | Counter |
|--------|---------|
| "I'll implement all slices then test everything at once" | A 23-commit branch with cross-slice bugs takes days to untangle. Per-slice verification catches issues when they're 1 commit old, not 15. |
| "This slice is too small to commit separately" | Small commits are rollback points. When slice 4 breaks, you revert to slice 3 — but only if slice 3 has its own commit. |
| "I need to build slice 3 to verify slice 2" | Then your slices are wrong. Re-slice so each unit is independently verifiable. |
| "The verification is obvious, I can see it works" | You can see it works NOW. When slice 5 breaks slice 2 silently, the only evidence is the verification you ran. |

## Red Flags

- More than 5 commits in a single slice → the slice is too big, split it
- Touching files outside the current slice's boundary → scope creep
- "I'll add the test later" → the test IS the verification, it can't be later
- New Pydantic model not in design.md → stop, update design first
- Committing without running verification → the entire BUILD process is broken

## Exit Criteria

All must be true:
- [ ] Every slice in scope.md is marked [x]
- [ ] Every slice has documented verification output
- [ ] Each slice has its own commit(s)
- [ ] No slice exceeds 5 commits
- [ ] All tests pass (if applicable)

## Next → /harden
