---
name: harden
description: "Tighten code quality and safety before PR. Trigger on: 'harden', 'harden this', 'pre-pr review', 'tighten up', 'check safety', 'LLM safety check', '하든', '단단하게', 'self review'"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent
argument-hint: ""
---

# HARDEN — Tighten before PR

## Overview

BUILD got the feature working. HARDEN makes it solid. This stage catches the class of bugs that "working code" hides: silent LLM fallbacks, dead imports from refactoring, missing edge cases in schema validation, and integration gaps between components.

## Entry Criteria

- All slices from /build are complete and verified
- All tests pass
- Code is committed (no uncommitted work mixed with hardening)

## Process

### Step 1: Integration test — does the full chain work?

Run the end-to-end flow, not just individual slices.

```bash
# Example: full reflect → feedback pipeline
pytest tests/integration/ -v

# Or manual E2E if no integration test exists:
# 1. Start services
# 2. Trigger the full pipeline
# 3. Verify final output matches expectation
```

If no integration test exists and the feature spans multiple components: write one now. This is not optional for multi-component features.

**Checkpoint**: The FULL path from entry to exit runs successfully, not just individual units.

### Step 2: LLM output safety audit

Scan all code touched in this branch for LLM-related safety issues:

```bash
# Find .get() with defaults on LLM output (silent data loss)
git diff main --unified=0 | grep -n '\.get('

# Find bare dict access without schema validation
git diff main --unified=0 | grep -n '\[.*\]' 

# Check all LLM calls use query_structured with Pydantic schema
grep -rn 'complete\|completion\|chat(' <changed_files>
```

Checklist:
- [ ] No `.get(key, default)` on LLM output — schema must enforce, not code
- [ ] All LLM responses validated through Pydantic `model_validate()`
- [ ] `model_validator` error messages are descriptive (they become reask prompts)
- [ ] No manual retry loops — `query_structured` handles retry
- [ ] `Field(ge=, le=)` or `Literal[...]` for all bounded values
- [ ] No `Optional` without explicit justification in design.md

**Checkpoint**: Every LLM output path has schema validation. Zero silent fallbacks.

### Step 3: Dead code cleanup — YOUR changes only

```bash
# Find imports that your changes made unused
git diff main --name-only | xargs grep -l 'import' | head -20

# Check for variables assigned but never read (in your diff only)
# Use ruff or pyflakes on changed files
ruff check $(git diff main --name-only -- '*.py') --select F841,F401
```

Rules:
- Remove imports YOUR changes made unused
- Remove variables YOUR changes made unused
- Do NOT touch pre-existing dead code (mention it, don't fix it)
- Do NOT "improve" adjacent code formatting or style

**Checkpoint**: `ruff check` passes on all changed files. Only YOUR orphans are cleaned.

### Step 4: Prompt context review (if LLM-involved feature)

For features that include LLM prompts:

- [ ] Prompt includes only necessary context (no token waste)
- [ ] Variable interpolation matches actual data shape
- [ ] Prompt and schema agree on output structure
- [ ] No contradictions between prompt instructions and schema constraints

Read each prompt template end-to-end. Ask: "If I were the LLM, would I know exactly what to produce?"

### Step 5: Commit hardening changes

```bash
ruff check --fix <changed_files> && ruff format <changed_files>
git add <specific files>
git commit -m "refactor: harden <feature> — <what was fixed>"
```

## Rationalizations

| Excuse | Counter |
|--------|---------|
| "Tests pass, it's fine" | Unit tests passed for wgdb-integration-v2 while .get() fallbacks silently swallowed bad LLM output. Passing tests prove the happy path, not safety. |
| "I'll clean up in the next PR" | No you won't. Dead imports from this PR will live for months. Clean YOUR mess now. |
| "LLM safety is overkill for this feature" | If the feature calls an LLM, the output is untrusted input. Period. One .get(key, "") turns a validation error (catchable) into wrong data (silent). |
| "Integration tests take too long to write" | A 10-line integration test catches the bugs that 50 unit tests miss. The time ratio is inverted. |

## Red Flags

- `.get()` with default on any LLM output → silent data loss
- `Optional` field added during BUILD that wasn't in design.md → unplanned optionality
- Integration test doesn't exist for multi-component feature → gaps between components are untested
- Hardening commit touches files outside the feature → scope creep in disguise
- "No issues found" without running the checklist → you didn't actually check

## Exit Criteria

All must be true:
- [ ] Integration test passes (or E2E manual verification documented)
- [ ] LLM safety checklist complete (if applicable)
- [ ] `ruff check` passes on all changed files
- [ ] No dead imports/variables from YOUR changes remain
- [ ] Prompt context reviewed (if applicable)
- [ ] Hardening changes committed

## References

→ `references/structured-output-checklist.md` for LLM safety patterns
→ `references/integration-test-checklist.md` for test patterns

## Next → /pr
