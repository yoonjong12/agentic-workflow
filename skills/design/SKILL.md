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

- `scope.md` exists (or scope is clearly defined)
- Slices are identified from /scope
- No implementation code written yet for this feature

## Process

### Step 1: Data flow — what goes in, what comes out

For each component or pipeline stage, define:
- Input: what data enters (type, source)
- Output: what data exits (type, destination)
- Side effects: what external state changes (DB writes, API calls)

```
reflect_pipeline:
  in:  OptimizationResult (from MEGA build step)
  out: CurationFeedback (to PCR /feedback endpoint)
  side: writes curation-learnings.json to session dir
```

Draw the flow as a simple chain:
```
[MEGA optimize] → OptimizationResult → [reflect] → ReflectionDraft → [LLM] → CurationFeedback → [PCR API]
```

**Checkpoint**: Every arrow has a named type. No "passes data to" without specifying what data.

### Step 2: Schema draft — Pydantic models

Write the core data models. Focus on:
- Field names and types (be precise — `list[str]` vs `list[WisdomRef]`)
- Required vs optional (default to required; optional needs justification)
- Validation rules as `model_validator` or `Field` constraints

```python
class CurationFeedback(BaseModel):
    session_id: str
    wisdom_ref: str  # W{N} format, resolved to hash before send
    effectiveness: Literal["effective", "ineffective", "neutral"]
    evidence: str  # one sentence, what happened when applied

    @model_validator(mode="after")
    def validate_wisdom_ref_format(self) -> Self:
        if not re.match(r"^W\d+$", self.wisdom_ref):
            raise ValueError(f"wisdom_ref must be W{{N}} format, got {self.wisdom_ref}")
        return self
```

**Checkpoint**: Could another developer implement the feature using ONLY these models and the data flow diagram?

### Step 3: Component boundaries — who calls whom

Define the interface between components:
- Which module/function owns each transformation
- Call direction (A calls B, never B calls A)
- Error contract (what happens when B fails — no silent fallbacks)

```
boundaries:
  meta_learner.py:
    - calls: reflect_pipeline.build_draft()
    - calls: pcr_client.send_feedback()
    - DOES NOT call: wisdom search (that's curator's job)

  reflect_pipeline.py:
    - called by: meta_learner
    - calls: LLM via litellm
    - returns: CurationFeedback or raises ReflectionError
```

**Checkpoint**: No circular dependencies. Every boundary has a clear owner.

### Step 4: LLM integration design (if applicable)

When the feature involves LLM calls, specify:
- Prompt → Schema mapping: which prompt produces which Pydantic model
- Validation strategy: what `model_validator` catches, what the prompt must enforce
- Retry behavior: handled by `query_structured` infrastructure (do NOT design manual retry)

```
LLM integration:
  prompt: reflection_prompt.md (takes: optimization_result, wisdom_context)
  schema: CurationFeedback
  validators:
    - wisdom_ref format (W{N})
    - effectiveness is valid Literal
    - evidence is non-empty
  retry: automatic via query_structured (schema errors become reask prompts)
```

**Checkpoint**: The schema validates everything the prompt might get wrong. The prompt only needs to be "good enough."

### Step 5: Write design.md

Create `design.md` in the working directory:

```markdown
# Design: <feature name>

## Data Flow
<chain diagram>

## Models
<Pydantic model definitions>

## Boundaries
<component → responsibility mapping>

## LLM Integration (if applicable)
<prompt → schema → validator mapping>

## Decisions
- <decision>: <chosen option> because <reason>

## Deferred
- <thing not designed now> — will address in <which slice>
```

## Rationalizations

| Excuse | Counter |
|--------|---------|
| "It's in my head, I'll just code it" | refactor/llm-output-safety-backup exists because the first implementation had .get() fallbacks everywhere. 7 commits to fix what design.md would have prevented. |
| "The design will change anyway" | Yes. A changed design doc is a 5-line edit. A changed implementation is a rewrite. |
| "I'm just adding a field" | Adding `PatternStep.gaps` required understanding the entire reflection pipeline flow. "Just a field" is never just a field. |
| "Pydantic models are self-documenting" | Models show WHAT. Design.md shows WHY this shape and WHERE it flows. |

## Red Flags

- Data flow has unnamed arrows ("passes to") → types not defined
- Schema has `Optional` without written justification → will become a silent None bug
- No boundaries section → everything will end up in one god-module
- LLM integration section missing when feature uses LLM → prompt/schema mismatch incoming
- design.md references code that doesn't exist yet → you're implementing, not designing

## Exit Criteria

All must be true:
- [ ] `design.md` exists with data flow, models, and boundaries
- [ ] Every data flow arrow has a named type
- [ ] Pydantic models have validation rules (not just field types)
- [ ] Component boundaries are defined with clear ownership
- [ ] If LLM involved: prompt→schema mapping documented
- [ ] Decisions section records non-obvious choices with reasons

## References

→ `references/structured-output-checklist.md` for Pydantic + LLM patterns

## Next → /build
