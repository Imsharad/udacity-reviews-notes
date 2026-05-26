# Edge-case guidance: indoor backup options

A weather-compatibility evaluator that just answers compatible/not-compatible on each activity in isolation leaves the planner without a fallback when the weather goes bad. The planner ends up with a day that has no approved activities, then either skips the day or picks the least-bad outdoor option.

The fix is to give the evaluator (and through it, the planner) explicit guidance about what to do when the primary activity fails the compatibility check. Indoor backups are the common case.

## Why it matters

A travel planner without backup-option guidance produces brittle itineraries. One thunderstorm day and the user has nothing to do. The planner either makes them stand in the rain or makes them sit in the hotel. Neither is the right answer.

The model can produce sensible fallbacks if you ask for them. Asked only for a binary compatibility decision, it produces only that, and the fallback logic never enters the prompt.

## The pattern

```python
ACTIVITY_AND_WEATHER_ARE_COMPATIBLE_SYSTEM_PROMPT = """
You are a weather-compatibility evaluator and backup-option suggester.

For each activity-weather pair, do two things:

1. Decide if the activity is safe and enjoyable in the given weather.
2. If not, name one or two indoor alternatives appropriate for the
   destination and the time of day.

## Output format

Return JSON with this exact shape:

{
  "decision": "IS_COMPATIBLE" | "IS_INCOMPATIBLE",
  "backup_suggestions": ["museum", "covered market"]   // empty if compatible
}

## Examples

Input: Outdoor hike in Rome, weather: thunderstorm.
Output:
{
  "decision": "IS_INCOMPATIBLE",
  "backup_suggestions": ["Vatican Museums", "Galleria Borghese"]
}

Input: Outdoor hike in Rome, weather: clear 72F.
Output:
{
  "decision": "IS_COMPATIBLE",
  "backup_suggestions": []
}
"""
```

The model is now doing two jobs in one call: classify, and suggest. The downstream planner reads the decision to decide whether to keep the activity, and reads the backups to fill the slot when the decision is IS_INCOMPATIBLE.

## When to put the fallback logic in the prompt vs in code

You have a choice. The prompt can produce fallbacks, or your planner code can have a static lookup table mapping destination + weather to backup categories.

- **Prompt-side fallbacks** are flexible (the model knows about specific museums in Rome) but vary by call.
- **Code-side fallbacks** are deterministic (always recommend the same backup) but blunt.

For a travel planner, prompt-side fallbacks usually win because the model has destination-specific knowledge a static table won't.

## Common pitfalls

- **No mention of backups at all**: the prompt asks only for compatibility. The planner has nothing to fill the slot with on a rainy day.
- **Backup logic as a separate prompt**: two LLM calls, one for compatibility, one for backups. Works but doubles latency and tokens.
- **Backups described in prose, not structured output**: the model returns "If it rains, you could go to the Vatican Museums." The planner can't parse that. Use the structured JSON shape.
- **Backups assumed available without checking**: the model suggests "the Louvre" for a destination that isn't Paris. Mitigate by passing the activities list to the evaluator so it knows what's locally available.

## Further reading

- [Anthropic — Be clear and direct](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#be-clear-and-direct) — covers explicit edge-case handling in prompts.
- [Anthropic — Chain prompts for stronger performance](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#chain-prompts) — when to split classify+suggest into two calls vs combine.
