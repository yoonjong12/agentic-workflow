# Structured Output Checklist

Pydantic + LLM output safety patterns. Use during /design (schema draft) and /harden (safety audit).

## Schema Design

- [ ] All LLM outputs are Pydantic `BaseModel` subclasses
- [ ] `Literal[...]` for any enum/category field — never `str`
- [ ] `Field(ge=, le=)` for bounded numeric values
- [ ] No `Optional` without documented reason (design.md Decisions section)
- [ ] `model_validator(mode="after")` for cross-field constraints
- [ ] Validator error messages are human-readable (they become LLM reask prompts)

## LLM Call Patterns

- [ ] Use `query_structured(prompt, Schema)` — one call does everything
- [ ] No manual retry loops — infrastructure handles retry via `_validate_and_retry_batch`
- [ ] No `.get(key, default)` on LLM output — schema must enforce, not code
- [ ] No `json.loads()` + manual parsing — use `model_validate()` only

## Dynamic Constraints

When validation depends on runtime state (valid indices, item counts):

```python
def _my_schema(n_items: int) -> type[MyModel]:
    class _Ctx(MyModel):
        @model_validator(mode="after")
        def _check(self) -> Self:
            if self.index >= n_items:
                raise ValueError(f"index {self.index} >= {n_items} items")
            return self
    return _Ctx

result = llm.query_structured(prompt, _my_schema(len(items)))
```

## Common Mistakes

| Mistake | Why it's bad | Fix |
|---------|-------------|-----|
| `.get("key", "")` | Empty string silently replaces missing data | Schema field + validator |
| `Optional[str] = None` | LLM omission becomes None, propagates silently | Make required, validate presence |
| `try/except` around parse | Swallows schema violations | Let `query_structured` handle retry |
| `str` for categorical field | LLM can invent categories | `Literal["a", "b", "c"]` |
| Manual JSON extraction | Fragile regex/string parsing | `response_format=Schema` in LLM call |
