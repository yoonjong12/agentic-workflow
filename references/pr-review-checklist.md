# PR Self-Review Checklist

Use during /pr Step 4 before submitting.

## Code Quality

- [ ] Every changed line traces to a slice in scope.md
- [ ] No debugging artifacts (print, console.log, commented code)
- [ ] No TODO without a ticket number
- [ ] No files that shouldn't be committed (.env, __pycache__, .DS_Store)
- [ ] Changed files match the feature boundary — no drive-by fixes

## Schema & LLM Safety

- [ ] No `.get(key, default)` on LLM output
- [ ] No new `Optional` fields without justification
- [ ] All `model_validator` messages are descriptive
- [ ] No manual retry loops around LLM calls
- [ ] Prompt and schema agree on output structure

## Commit Hygiene

- [ ] Each commit maps to a slice or hardening step
- [ ] Commit messages use conventional format (feat:, fix:, refactor:, test:)
- [ ] No "wip", "fix stuff", "update" messages
- [ ] No merge commits from main (rebase if needed, before push)

## PR Body

- [ ] Title under 70 characters, describes the feature
- [ ] Summary explains WHY, not just WHAT
- [ ] Changes section maps to slices with verification results
- [ ] Design decisions documented for non-obvious choices
- [ ] Test results are actual output, not "tests pass"
- [ ] "Not included" section lists scope.md non-goals

## Security

- [ ] No hardcoded secrets, API keys, or credentials in diff
- [ ] No new dependencies without justification
- [ ] Environment variables used for sensitive config
