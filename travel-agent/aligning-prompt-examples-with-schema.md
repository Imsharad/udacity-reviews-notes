# Aligning prompt examples with your schema

When you embed a JSON example in a prompt to show the LLM what shape to produce, that example becomes the ground truth in the model's eyes — not your Pydantic model. If the example uses `destination` but your model expects `city`, the LLM will produce `destination`, and `model_validate_json` will raise.

## Why it matters

Few-shot examples are the most effective way to steer output format. They're also the most reliable way to *misdirect* the model when they drift from the actual validation schema. The LLM will trust the example over abstract field descriptions every time.

The drift usually creeps in this way: you write the Pydantic model first, draft a prompt with a hand-written JSON example that approximates it, then later refactor the Pydantic model (rename a field, change a type, add a required field) without going back to update the prompt example. The model still validates, the example still parses as JSON, but they no longer agree.

When the LLM is asked to produce output, it copies the example. Validation against the updated Pydantic model fails. The error message points at the LLM as the culprit ("missing required field `start_date`") when the actual culprit is your stale prompt.

## The pattern

The cleanest fix: stop hand-writing the example. Generate it from the Pydantic model itself.

```python
from pydantic import BaseModel

class TravelPlan(BaseModel):
    city: str
    start_date: str
    end_date: str
    total_cost: float
    itinerary_days: list[ItineraryDay]

ITINERARY_AGENT_SYSTEM_PROMPT = f"""
You are an expert travel planner.

Your output must validate against this schema:

{TravelPlan.model_json_schema()}

Example of a valid output:
{TravelPlan(
    city="AgentsVille",
    start_date="2025-06-10",
    end_date="2025-06-13",
    total_cost=420.0,
    itinerary_days=[...],
).model_dump_json(indent=2)}
"""
```

Two things happen with this approach:

1. The schema in the prompt **always matches** the schema in the code — they share a source.
2. The example is a real, validated `TravelPlan` instance — there's no way for the example to drift from the model without breaking the prompt at import time.

When the model is refactored (add a field, rename one, change a type), the prompt updates automatically.

## Common pitfalls

- **Hand-written JSON example uses different field names than the Pydantic model**: `destination` vs `city`, `dates` vs `start_date`/`end_date`, `daily_plan` vs `itinerary_days`. The model copies the example field names; validation fails.
- **Example omits required fields**: prompt shows `{"city": "AgentsVille", "itinerary_days": [...]}` and the model omits `start_date`/`end_date` because they weren't in the example.
- **Nested types shown as plain strings**: example shows `"weather": "sunny"` but the model expects a `Weather` object with `temperature` and `precipitation_probability` fields.
- **Schema mentioned by name but not embedded**: "Your output should match the TravelPlan model" — the LLM has no way to know what TravelPlan looks like unless the schema is actually in the prompt text.

## Further reading

- [Pydantic — JSON Schema](https://docs.pydantic.dev/latest/concepts/json_schema/) — `model_json_schema()` and how to inject it into prompts.
- [OpenAI Cookbook — Structured Outputs introduction](https://developers.openai.com/cookbook/examples/structured_outputs_intro) — covers Pydantic-model alignment with the structured-output API.
- [Anthropic — Increase output consistency](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering) — guidance on using examples to anchor output structure.
