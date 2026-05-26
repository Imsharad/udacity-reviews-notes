# Writing a real system prompt from scratch

Starter notebooks ship with placeholder prompts. They look like prompts. They're not. A line like `ITINERARY_AGENT_SYSTEM_PROMPT = "Itinerary agent prompt"` or a triple-quoted block of `***********` is a TODO marker, not an instruction the model can act on.

A prompt left as starter scaffolding will not be detected by tests, will not be flagged by type-checkers, and will produce *something* when called — usually low-quality, sometimes unsafe. The agent silently degrades.

## Why it matters

The model is robust to bad prompts. It will produce an itinerary even if you ask "do the thing." That robustness hides the regression: the placeholder prompt's output looks plausible enough to pass a casual eyeball check, but it will fail any structured evaluation. Output schema violations, hallucinated activity IDs, ignored weather constraints — these are the symptoms of a model running without real instructions.

The fix is not to tweak the placeholder. The fix is to delete it and write the real prompt.

## The pattern

A working system prompt for the itinerary agent has four blocks, in this order:

```python
ITINERARY_AGENT_SYSTEM_PROMPT = f"""
You are an expert travel planner who builds day-by-day itineraries
respecting traveler interests, weather, and budget.

## Trip context

Destination: {vacation_info.destination}
Dates: {vacation_info.date_of_arrival} to {vacation_info.date_of_departure}
Travelers: {vacation_info.travelers}
Interests: {", ".join(vacation_info.interests)}
Budget: {vacation_info.budget_usd} USD

Weather forecast:
{weather_for_dates_df.to_markdown(index=False)}

Available activities:
{activities_for_dates_df.to_markdown(index=False)}

## How to plan

1. For each day, read the weather for that date.
2. Filter activities by weather compatibility and traveler interests.
3. Pick activities within the running budget.
4. Validate the assembled plan against the schema before emitting.

## Output

Return a single JSON object matching this schema:

{TravelPlan.model_json_schema()}

Output the JSON only. No prose, no reasoning trace, no markdown fences.
"""
```

**Role** (one or two sentences). **Context** (the actual data via f-string substitution). **Procedure** (numbered steps). **Output contract** (schema + format constraints). Anything else is decoration.

## Common pitfalls

- **Leaving the placeholder**: `"Itinerary agent prompt"`, `"**********"`, `"<- Fill in ->"` — all variations of "I will replace this later." None get replaced because the notebook runs and produces output, so the TODO never reasserts itself.
- **Treating the prompt as a comment**: writing one big sentence describing what the model should do, with no structure. The model handles it badly because there's no signal about ordering or output format.
- **Copying a prompt from a different project verbatim**: the role line still says "video game research" or "code reviewer" because the notebook started as a fork. The travel-planner part lives only in the variable name.
- **Skipping the output contract**: the prompt explains how to plan but never says how to format the result. The model picks a format, sometimes the right one, sometimes plain text wrapped in prose.
- **Embedding instructions only in the user message**: the system prompt is empty or generic, and all the load-bearing context lives in the runtime user message. This works but makes the agent hard to reason about and impossible to cache.

## Further reading

- [Anthropic — Define your task and audience](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#define-your-task-and-audience) — first-party on starting from a real specification rather than a placeholder.
- [Anthropic — Be clear and direct](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#be-clear-and-direct) — companion section on explicit task and output format.
- [OpenAI Cookbook — Prompt patterns](https://developers.openai.com/cookbook/) — recipes that show role-context-procedure-output as the canonical structure.
