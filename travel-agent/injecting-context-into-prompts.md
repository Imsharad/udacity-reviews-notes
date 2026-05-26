# Injecting context into prompts

The prompt template and the data live in different places. The template names placeholders; the runtime fills them. A common failure mode is a prompt that *names* the data it needs (weather, activities, VacationInfo) but never actually substitutes it — the model sees the placeholders, not the values.

This is mostly a Python f-string discipline problem, with a couple of structural traps.

## Why it matters

A travel planner that doesn't see the weather data plans for the wrong weather. A planner that doesn't see the activities data invents activity IDs that don't exist. A planner that doesn't see VacationInfo guesses at the destination. All of these failures are silent — the model produces fluent output, the downstream JSON parser accepts it, the validator catches the hallucinated IDs only after the plan reaches the user.

The fix is not "tell the model harder." The fix is to make sure the real values land inside the prompt before it hits the model.

## The pattern

```python
ITINERARY_AGENT_SYSTEM_PROMPT = f"""
You are an expert travel planner.

## Trip context

Destination: {vacation_info.destination}
Dates: {vacation_info.date_of_arrival} to {vacation_info.date_of_departure}
Travelers: {vacation_info.travelers}
Interests: {", ".join(vacation_info.interests)}
Budget: {vacation_info.budget_usd} USD

## Weather forecast for the trip dates

{weather_for_dates_df.to_markdown(index=False)}

## Activities available on those dates

{activities_for_dates_df.to_markdown(index=False)}

## Output schema

{TravelPlan.model_json_schema()}
"""
```

Three things are doing work here:

1. **f-string substitution** at prompt construction time. The `f"""..."""` evaluates `{vacation_info.destination}` etc. once, against live Python values.
2. **Tabular rendering** for the dataframes. `to_markdown()` produces a table the model reads naturally. Raw `repr(df)` works in a pinch but is less readable.
3. **Schema injection** for the output contract. `{TravelPlan.model_json_schema()}` keeps the prompt and the Pydantic model in lockstep — change the model, the prompt updates on the next run.

Verify before sending: `print(ITINERARY_AGENT_SYSTEM_PROMPT[:2000])`. If you see `{vacation_info...` in the output, the f-string never evaluated.

## Common pitfalls

- **Triple-quoted string instead of f-string**: dropping the `f` prefix turns the placeholders into literal text. The prompt names the data but the model never sees it. Symptom: model invents details that should have been provided.
- **Unfilled scaffold markers**: the starter notebook had `*********` or `<-- Specify the context -->` as a placeholder for the data block. Easy to leave in, hard to spot.
- **Wrong variable name**: `{vacation_info}` instead of `{vacation_info.destination}` — Python embeds the repr of the whole object, which often dumps as `VacationInfo(...)` with curly braces that confuse downstream parsing.
- **Dataframes never filtered for the trip dates**: passing `weather_df` (all dates) instead of `weather_for_dates_df` (just the trip window). The model wastes attention on irrelevant rows and sometimes plans for the wrong week.
- **Schema referenced by name only**: "Output must match TravelPlan." The model has no idea what TravelPlan is. Embed `model_json_schema()` directly.

## Further reading

- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — discusses context engineering as the load-bearing skill for agentic systems.
- [Pydantic — model_json_schema()](https://docs.pydantic.dev/latest/concepts/json_schema/) — first-party reference for the schema injection pattern.
- [Python docs — Formatted string literals](https://docs.python.org/3/reference/lexical_analysis.html#f-strings) — the actual semantics of `f""` and how attribute access works inside braces.
