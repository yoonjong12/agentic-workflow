---
name: pr
description: "Self-review and create a pull request. Trigger on: 'pr', 'create pr', 'pull request', 'submit pr', 'open pr', 'PR 올려', 'PR 작성', 'self review'"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
argument-hint: "<base branch, default: main>"
---

# PR — Self-review, then submit

## Overview

PR is the final gate. Read your own diff as a reviewer would, write a PR that explains WHY not WHAT, and submit. The diff shows what changed — the PR body explains what the reviewer can't see from code alone.

## Entry Criteria

- /harden complete
- All tests pass
- All changes committed
- Branch is pushed to remote

## Session Resolution

Read `.claude/workflows/.active` → `<session-dir> = .claude/workflows/<id>/`. PR pulls its narrative material from `<session-dir>/scope.md` (Summary, Changes, Not included) and `<session-dir>/design.md` (Design decisions). If `.active` is missing, ask the user which session is being shipped.

## Process

### Step 1: Read the full diff

```bash
git diff main...HEAD --stat
git diff main...HEAD
```

Read every changed line. Not skim — read. You just spent hours in this code; a reviewer is seeing it for the first time.

While reading, check:
- [ ] Every changed line traces to a slice in `<session-dir>/scope.md`
- [ ] No debugging artifacts (print statements, commented code, TODO without ticket)
- [ ] No files that shouldn't be committed (.env, local config, __pycache__)
- [ ] Commit history tells a coherent story (slice-by-slice, not random)

**Checkpoint**: You can explain why every changed file changed.

### Step 2: Organize commits

Review the commit history:

```bash
git log main..HEAD --oneline
```

Each commit should map to a slice or a hardening step. If commits are messy:
- Interactive rebase to squash fixup commits (only if not yet pushed)
- If already pushed, leave as-is — force-push destroys review context

Commit message quality check:
- [ ] Each message starts with type: `feat:`, `fix:`, `refactor:`, `test:`
- [ ] Message describes WHAT changed, not HOW
- [ ] No generic messages ("fix stuff", "wip", "update")

### Step 3: Write PR body

Structure:

```markdown
## Summary
<1-3 sentences: what this PR does and WHY>

## Changes
<bullet list mapping to slices>
- Slice 1: <what> — <verified how>
- Slice 2: <what> — <verified how>
- ...

## Design decisions
<non-obvious choices and their reasons — from <session-dir>/design.md>

## Test results
<paste actual test output or summary>
- Unit: X passed
- Integration: X passed
- Manual: <what you verified>

## Not included (intentional)
<from <session-dir>/scope.md non-goals — helps reviewer not ask "why didn't you also...">
```

**Checkpoint**: A reviewer reading only the PR body (not the code) understands the scope, the decisions, and what was tested.

### Step 4: Self-review checklist

Before submitting, one final pass:

- [ ] PR title is under 70 characters, describes the feature (not the task)
- [ ] Summary explains WHY, not just WHAT
- [ ] "Not included" section prevents scope questions
- [ ] Test results are actual output, not "tests pass"
- [ ] No secrets in diff (.env, API keys, credentials)
- [ ] Base branch is correct

### Step 5: Create PR

```bash
gh pr create --title "<title>" --body "$(cat <<'EOF'
<PR body from Step 3>
EOF
)"
```

## Rationalizations

| Excuse | Counter |
|--------|---------|
| "I just wrote the code, I don't need to re-read it" | You wrote it in slices over hours. The diff is the FIRST time you see all changes together. Bugs hide at slice boundaries. |
| "The commits tell the story" | Commits tell WHAT happened. The PR body tells WHY and WHAT WAS EXCLUDED. Reviewers need both. |
| "Non-goals section is redundant" | "Why didn't you also add X?" is the #1 review comment on PRs without non-goals. Save everyone's time. |
| "I'll fix the commit messages later" | You won't. And messy history makes `git bisect` useless when debugging in 3 months. |

## Red Flags

- PR body is just a title → reviewer has zero context
- No test results section → "trust me it works"
- Diff contains files not mentioned in any slice → scope creep leaked through
- PR has 20+ changed files → should this be multiple PRs?
- No "Not included" section → first review comment will be "why didn't you also..."

## Exit Criteria

All must be true:
- [ ] Full diff read and self-reviewed
- [ ] PR body has Summary, Changes, Design decisions, Test results, Not included
- [ ] Self-review checklist passed
- [ ] PR created and URL obtained
- [ ] `<session-dir>/scope.md` and `<session-dir>/design.md` are committed (the session directory itself persists for reference)

## Done. PR submitted.
