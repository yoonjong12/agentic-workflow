# Integration Test Checklist

Patterns for testing multi-component features. Use during /harden.

## When to Write Integration Tests

- Feature spans 2+ components (e.g., MEGA → PCR)
- Data flows through LLM (input → prompt → schema → output)
- Pipeline has multiple stages (reflect → feedback → accumulate)

Single-component, no-LLM features: unit tests are sufficient.

## Structure

```python
class TestFeatureIntegration:
    """Test the full pipeline: <entry> → <exit>"""

    def test_happy_path(self):
        """Full chain with valid input produces expected output."""
        # Arrange: set up all components with known input
        # Act: trigger the entry point
        # Assert: verify the final output AND intermediate state

    def test_component_boundary(self):
        """Data crosses component boundary with correct shape."""
        # Verify the OUTPUT of component A matches the INPUT contract of component B

    def test_error_propagation(self):
        """Errors from downstream components surface correctly."""
        # Verify errors are raised, not swallowed
```

## Key Principles

- [ ] Test the FULL path, not just individual units
- [ ] Use real dependencies where practical (real DB, real file system)
- [ ] Mock only external services you don't control (third-party APIs)
- [ ] Assert on final output AND intermediate state at component boundaries
- [ ] Test error propagation: downstream failure must surface, not silently pass

## LLM Integration Tests

For features with LLM calls:

- [ ] Test with a known prompt→schema pair that has a deterministic expected shape
- [ ] Verify schema validation catches intentionally bad LLM output
- [ ] Test `model_validator` with edge case inputs
- [ ] Do NOT test LLM quality (non-deterministic) — test the validation pipeline

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Mocking the database | Use a real test database — mock/prod divergence masks bugs |
| Asserting only final output | Also assert intermediate state at component boundaries |
| No error path tests | Test what happens when a downstream component fails |
| Testing LLM output quality | Test the validation pipeline, not LLM creativity |
