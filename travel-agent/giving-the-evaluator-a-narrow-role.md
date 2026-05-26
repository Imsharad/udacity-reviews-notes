# Giving the evaluator a narrow role

A planner has a wide role: "expert travel planner." An evaluator should have a narrow one: "weather-compatibility evaluator." The narrowness is load-bearing.

This is a sibling of [giving-the-agent-a-clear-role](./giving-the-agent-a-clear-role.md), but the failure mode is different. The agent without a role hedges and produces fluent generic output. The evaluator without a role *also* hedges, but for a classifier that has to return one of two strings, hedging is fatal.

## Why it matters

A model told "you are a helpful assistant" and asked whether an outdoor picnic is compatible with a thunderstorm will respond with a paragraph. "While picnics in thunderstorms are not ideal, with proper preparation it may still be feasible. However, you may want to consider..." Polite, fluent, completely useless to a parser checking for IS_COMPATIBLE or IS_INCOMPATIBLE.

A model told "you are a weather-compatibility evaluator" and asked the same question returns "IS_INCOMPATIBLE." The narrow role aligns the response style with the binary output contract.

The narrowness also helps with edge cases. A "weather-compatibility evaluator" knows it does not handle scheduling, packing advice, or alternative suggestions unless asked. A "travel assistant" handed the same input might wander into those territories.

## The pattern

```python
ACTIVITY_AND_WEATHER_ARE_COMPATIBLE_SYSTEM_PROMPT = """
You are a weather-compatibility evaluator. Your only job is to decide
whether a planned activity is safe and enjoyable in a given weather
forecast.

You do not suggest alternatives. You do not explain your reasoning in
the output. You do not respond to questions outside your scope.

Output exactly one of these two strings, with nothing before or after:
- IS_COMPATIBLE
- IS_INCOMPATIBLE
"""
```

Three things narrow the role:

1. **"Your only job is to decide..."** — explicit single-task scope.
2. **"You do not suggest alternatives. You do not explain reasoning."** — explicit out-of-scope list.
3. **The output contract is one of two strings** — the role and the output reinforce each other.

If you do want the evaluator to *also* suggest backups, give it a narrow role for that wider job ("compatibility evaluator and backup suggester"), not a wide role.

## When the role is too narrow

Narrow roles have a failure mode: the model refuses inputs that *are* in scope but don't match the example shape. A "weather-compatibility evaluator" given an indoor activity might return "IS_COMPATIBLE" reflexively without checking; or worse, refuse to answer because indoor activities don't have weather requirements.

Mitigate by including in-scope edge cases as examples. One example with an indoor activity + bad weather labeled IS_COMPATIBLE tells the model that indoor activities are in scope and uniformly compatible.

## Common pitfalls

- **Generic assistant role**: "You are a helpful assistant." The model defaults to verbose, helpful prose. Bad for classifier outputs.
- **Travel-planner role on a classifier prompt**: copying the planner's role over to the evaluator. The role mismatch produces planner-style output (itinerary suggestions) when you want classifier output.
- **Two competing roles**: "You are a weather expert and a helpful planning assistant." The carrier-wave role (assistant) wins; the specialist role gets diluted.
- **No explicit "do not" list**: the role is narrow but the prompt never says what the model should refuse to do. Symptom: the model still wanders into alternative suggestions and explanations.

## Further reading

- [Giving the agent a clear role](./giving-the-agent-a-clear-role.md) — the wider-role sibling page in this repo.
- [Anthropic — Give Claude a role](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#give-claude-a-role) — first-party guidance on role assignment, with examples of narrow vs broad roles.
