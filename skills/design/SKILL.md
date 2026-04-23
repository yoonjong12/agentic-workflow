---
name: design
description: "Schema-first design before implementation. Trigger on: 'design', 'design this', 'schema design', 'draw the architecture', 'define the interface', 'data flow', '설계', '디자인', 'model first', 'pydantic model'"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent
argument-hint: "<feature or component to design>"
---

# DESIGN — Schema first, code later

## Overview

Design is where you decide the shape of data and the boundaries between components BEFORE writing implementation code. The deliverable is a design document with schemas, data flow, and component boundaries — not code.

## Entry Criteria

- `<session-dir>/scope.md` exists (or scope is clearly defined)
- Slices are identified from /scope
- No implementation code written yet for this feature

## Session Resolution

Read `.claude/workflows/.active` → `<session-dir> = .claude/workflows/<id>/`. If `.active` is missing, list existing sessions and ask the user to attach or create one (same protocol as /scope). Read `<session-dir>/scope.md`; write `<session-dir>/design.md`.

## Process

### Step 0: Anchor — define the target state from Acceptance

Before reading any code, establish what "done" looks like:

1. Read `<session-dir>/scope.md` — the Acceptance is the target state.
2. Collect references that inform the target: design docs, specs, API contracts, user requirements.
3. Define the target state in concrete terms (expected outputs, expected behavior, expected data shapes).

**References (design docs, specs) and existing code are BOTH inputs — neither is the target.** The Acceptance is the target. References inform it, code reveals the current gap.

**Anti-pattern**: Reading existing code first → designing "current + small delta." This anchors to implementation instead of the goal.

**Checkpoint**: Can you state the target state without referencing any source file or function name? If not, you're anchored to code.

### Step 1: Data flow — what goes in, what comes out

For each component or pipeline stage, define:
- Input: what data enters (type, source)
- Output: what data exits (type, destination)
- Side effects: what external state changes (DB writes, API calls)

```
notification_service:
  in:  Event (from event bus)
  out: DeliveryResult (to audit log)
  side: sends notification via selected channel
```

Draw the flow as a simple chain:
```
[Event Bus] → Event → [preference lookup] → UserPreference → [channel router] → DeliveryResult → [Audit Log]
```

**Checkpoint**: Every arrow has a named type. No "passes data to" without specifying what data.

### Step 2: Schema draft — Pydantic models

Write the core data models. Focus on:
- Field names and types (be precise — `list[str]` vs `list[WisdomRef]`)
- Required vs optional (default to required; optional needs justification)
- Validation rules as `model_validator` or `Field` constraints

```python
class UserPreference(BaseModel):
    user_id: str
    channels: list[Literal["email", "slack", "sms"]]
    quiet_hours: tuple[int, int]  # (start_hour, end_hour) in UTC

    @model_validator(mode="after")
    def validate_quiet_hours(self) -> Self:
        start, end = self.quiet_hours
        if not (0 <= start <= 23 and 0 <= end <= 23):
            raise ValueError(f"quiet_hours must be 0-23, got ({start}, {end})")
        return self
```

**Checkpoint**: Could another developer implement the feature using ONLY these models and the data flow diagram?

### Step 3: Gap analysis — current state vs target state

Now read the existing code. Compare with the target state from Step 0:
- What already exists and matches the target?
- What exists but diverges from the target?
- What's missing entirely?

Design only the changes needed to close the gap. Do not redesign what already works correctly.

**Checkpoint**: Every item in your design traces to a gap. No "improvements" that aren't gaps.

### Step 4: Component boundaries — who calls whom

Define the interface between components:
- Which module/function owns each transformation
- Call direction (A calls B, never B calls A)
- Error contract (what happens when B fails — no silent fallbacks)

```
boundaries:
  notification_service.py:
    - calls: preference_store.get(user_id)
    - calls: channel_router.send(preference, event)
    - DOES NOT call: event bus directly (receives events via handler)

  channel_router.py:
    - called by: notification_service
    - calls: email/slack/sms adapters
    - returns: DeliveryResult or raises ChannelError
```

**Checkpoint**: No circular dependencies. Every boundary has a clear owner.

### Step 5: LLM integration design (if applicable)

When the feature involves LLM calls, specify:
- Prompt → Schema mapping: which prompt produces which Pydantic model
- Validation strategy: what `model_validator` catches, what the prompt must enforce
- Retry behavior: handled by `query_structured` infrastructure (do NOT design manual retry)

```
LLM integration:
  prompt: classification_prompt.md (takes: event_payload, user_history)
  schema: EventClassification
  validators:
    - priority is valid Literal["urgent", "normal", "low"]
    - category matches known categories
    - summary is non-empty
  retry: automatic via query_structured (schema errors become reask prompts)
```

**Checkpoint**: The schema validates everything the prompt might get wrong. The prompt only needs to be "good enough."

### Step 6: Write design.md

Create `<session-dir>/design.md` (resolved in Session Resolution above):

```markdown
# Design: <feature name>

## Target State
<what "done" looks like — from Acceptance, not from code>

## Data Flow
<chain diagram>

## Models
<Pydantic model definitions>

## Gap Analysis
<current vs target — what changes>

## Boundaries
<component → responsibility mapping>

## LLM Integration (if applicable)
<prompt → schema → validator mapping>

## Change Class (per file)
- `path/to/file.py`: **additive** | **destructive** — <one line why>
- destructive changes must list the specific existing behavior being removed/replaced

## Risks
- <what could break, who notices, how to catch>

## Decisions
- <decision>: <chosen option> because <reason>

## Deferred
- <thing not designed now> — will address in <which slice>
```

### Step 7: Scope coverage check

The design must explain every slice in scope.md. Walk through each slice and verify:

1. Read `<session-dir>/scope.md` — list every slice and the Acceptance criteria.
2. For each slice, confirm the design answers:
   - **What data** does this slice produce/consume? → must appear in Data Flow or Models
   - **What component** owns this slice's logic? → must appear in Boundaries
   - **What decisions** does this slice require? → must appear in Decisions or be self-evident from the design
3. For the Acceptance scenario specifically: trace the full path through the design. Every step the user will perform must have a designed component behind it.

Present the coverage matrix:

```
Slice 1: Domain model          → Models §Domain, §Benchmark ✓
Slice 2: HF loader             → Data Flow §HF Dataset, Boundaries §datasets.py ✓
Slice 3: PCR extraction        → LLM Integration §pcr_prompt, Models §PCRTriple ✓
Slice 4: Seeder embedding      → ??? ✗ — design doesn't specify embedding source
```

If any slice has `✗`, stop. Add the missing design section before proceeding to /build.

**Checkpoint**: Every scope slice maps to at least one design section. No slice requires improvisation during build.

### Step 8: Approval Gate

Design is a contract with the user. /build is blocked until the user explicitly approves THIS version of design.md.

1. Present a summary of the design to the user and ask: **"Approve design.md? Reply 'approved' to unlock /build."**
2. Only after the user replies literally `approved` (not "looks good", "ok", "sure"), record the hash of design.md:

   ```bash
   sha256sum "<session-dir>/design.md" | awk '{print $1}' > "<session-dir>/.design-approved"
   ```

3. If design.md changes after approval, /build will detect the hash mismatch and refuse to run. Re-run /design to re-approve.

**Authenticity vs freshness**: This gate enforces *freshness* (design.md has not changed since approval). It does NOT enforce *authenticity* — the rule "only write the hash after user literally typed 'approved'" lives in this skill's prompt and is honored by the model, not cryptographically. Do not bypass by writing the hash yourself.

**Gitignore**: `<session-dir>/.design-approved` is per-developer state. Ensure `.claude/workflows/*/.design-approved` is in `.gitignore` for the project.

## Rationalizations

| Excuse | Counter |
|--------|---------|
| "It's in my head, I'll just code it" | Every "safety refactor" branch exists because the first implementation had .get() fallbacks everywhere. 7 commits to fix what design.md would have prevented. |
| "The design will change anyway" | Yes. A changed design doc is a 5-line edit. A changed implementation is a rewrite. |
| "I'm just adding a field" | Adding a single field often requires understanding the entire pipeline that consumes it. "Just a field" is never just a field. |
| "Pydantic models are self-documenting" | Models show WHAT. Design.md shows WHY this shape and WHERE it flows. |

## Red Flags

- Data flow has unnamed arrows ("passes to") → types not defined
- Schema has `Optional` without written justification → will become a silent None bug
- No boundaries section → everything will end up in one god-module
- LLM integration section missing when feature uses LLM → prompt/schema mismatch incoming
- design.md references code that doesn't exist yet → you're implementing, not designing
- Scope slice has no corresponding design section → will require ad-hoc decisions during build
- Build starts with `✗` in coverage matrix → design incomplete, stop and fill gaps first

## Exit Criteria

All must be true:
- [ ] `<session-dir>/design.md` exists with data flow, models, and boundaries
- [ ] Every data flow arrow has a named type
- [ ] Pydantic models have validation rules (not just field types)
- [ ] Component boundaries are defined with clear ownership
- [ ] If LLM involved: prompt→schema mapping documented
- [ ] Decisions section records non-obvious choices with reasons
- [ ] **Scope coverage**: every slice in scope.md maps to at least one design section (Step 7 passed)
- [ ] Change Class declared per file (additive | destructive with justification)
- [ ] Risks section lists concrete failure modes
- [ ] User replied literally `approved` AND `<session-dir>/.design-approved` contains the current sha256 of design.md (Step 8 passed)

## References

→ `references/structured-output-checklist.md` for Pydantic + LLM patterns

## Next → /build
