# Chain-of-thought in system prompts

A model asked to produce a day-by-day travel itinerary in a single forward pass will produce something that *looks* like a plan but ignores the constraints. The same model, asked to reason day-by-day before producing the final JSON, will catch its own mistakes: rainy day for an outdoor activity, budget overrun by Tuesday, an interest left untouched.

Chain-of-thought (CoT) is the scaffolding that creates that second behavior. In a system prompt, it lives between the role line and the output contract.

## Why it matters

Without explicit reasoning steps, the model treats itinerary generation as a single shape-matching task: produce JSON that looks like a TravelPlan. With reasoning steps, it treats the same task as a sequence of small decisions: for each day, look up the weather, filter activities by interest, check budget, then commit.

The cost of skipping CoT is silent. The itinerary looks valid. The schema matches. But on inspection, the model picked an outdoor hike on a thunderstorm day, or scheduled three museum days for a traveler who said "outdoors." CoT surfaces those decisions where you can read them.

In long-running agent loops, the cost compounds. A planning agent without CoT produces a plan that a downstream evaluator then rejects, which triggers a revision cycle, which costs latency and tokens. A planning agent with CoT often gets the constraints right on the first pass.

## The pattern

```python
ITINERARY_AGENT_SYSTEM_PROMPT = f"""
You are an expert travel planner. You build day-by-day itineraries that
respect the traveler's interests, the weather forecast, and the
destination's available activities.

## How to plan (think step by step)

For each day of the trip:

1. **Read the weather forecast** for that date from the provided weather
   data. Note temperature, precipitation, wind.
2. **Filter the candidate activities** for that date to ones whose
   `weather_requirements` are compatible with step 1.
3. **From the filtered set, pick activities that match** the traveler's
   stated interests in VacationInfo.interests.
4. **Check the running budget**. If the activities selected so far
   exceed the budget, drop the lowest-interest item.
5. **Record the day's plan** with date, activities, and rationale.

After all days are planned, **validate the full itinerary** against
the schema below before emitting.

## Output schema

{TravelPlan.model_json_schema()}

Your output is the final JSON object only. Do not include the reasoning
trace in the output.
"""
```

The reasoning steps are *in the prompt*, not in the output. The model uses them internally; the output is still the structured JSON the downstream parser expects.

## Common pitfalls

- **No reasoning steps at all**: the prompt jumps from the role line to the schema. Symptoms: itineraries that ignore weather, repeat activities, or violate budget.
- **Steps named but not numbered**: "Think about the weather and the activities and the budget." The model treats this as one diffuse instruction, not a sequence. Numbering is load-bearing.
- **Reasoning instructed but no separation from output**: the model emits the reasoning trace inside the JSON. The fix is to be explicit: "Do not include the reasoning trace in the output."
- **CoT as a single line at the end**: "Think step-by-step." On its own this helps the model marginally, but a numbered procedure for the specific task helps far more.

## Further reading

- [Anthropic — Manual chain of thought](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#manual-chain-of-thought-as-a-fallback) — first-party section on explicit reasoning steps in prompts.
- [Prompting Guide — Chain-of-Thought Prompting](https://www.promptingguide.ai/techniques/cot) — broader survey including zero-shot and few-shot CoT.
- [Wei et al. 2022 — Chain-of-Thought Prompting Elicits Reasoning in LLMs](https://arxiv.org/abs/2201.11903) — the original paper, useful background on why this works.
