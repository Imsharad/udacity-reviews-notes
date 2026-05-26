# Few-shot prompting: both sides matter

A binary classifier prompt that ships with three IS_COMPATIBLE examples and zero IS_INCOMPATIBLE examples will skew toward IS_COMPATIBLE in production. The model learns the format and the bias at the same time.

This is the most common subtle mistake in few-shot prompting for classifiers. You can fix it with one extra example.

## Why it matters

The examples in a prompt do two things: they teach the format, and they teach the prior. If every example shows the positive class, the model concludes the positive class is the default. When the input is genuinely ambiguous, the model leans positive.

For a weather-compatibility evaluator, this means outdoor hikes get approved in drizzle, beach days get approved in 18C wind, and the planner downstream keeps activities that should be replaced. The bug is in the prompt's example distribution.

## The pattern

```python
ACTIVITY_AND_WEATHER_ARE_COMPATIBLE_SYSTEM_PROMPT = """
You are a weather-compatibility evaluator.

## Output

Exactly one of: IS_COMPATIBLE, IS_INCOMPATIBLE.

## Examples

Input: Hike, weather: clear, 72F.
Output: IS_COMPATIBLE

Input: Beach day, weather: clear, 82F, light breeze.
Output: IS_COMPATIBLE

Input: Outdoor concert, weather: thunderstorm, 60F.
Output: IS_INCOMPATIBLE

Input: Open-air market, weather: light rain, 65F.
Output: IS_INCOMPATIBLE

Input: Indoor museum tour, weather: heavy snow, 25F.
Output: IS_COMPATIBLE
"""
```

Five examples, balanced across both classes, with at least one "near-miss" case in each (light rain, indoor activity in bad weather). This covers the format, the prior, and the edge cases.

A useful heuristic: if you can imagine the model picking the wrong label on an input, write that input as one of the examples with the correct label.

## How many examples

For a binary classifier with a clear decision boundary, three to five examples is usually enough. More than seven and the model starts overfitting to the example distribution (e.g., always picking the more common label even when the input is unrelated).

If the classifier has a complex decision boundary (e.g., "compatible if temperature is in range AND precipitation is below threshold AND wind is below threshold"), use more examples, and include the boundary cases on purpose.

## Common pitfalls

- **One-sided examples**: only IS_COMPATIBLE shown. Model defaults to IS_COMPATIBLE on ambiguity.
- **Examples without the format**: the example shows the reasoning but not the exact output string. Model picks its own format.
- **Examples that disagree with the rule**: the rule says "outdoor activity in rain is incompatible," but one example shows an outdoor activity in light rain labeled IS_COMPATIBLE without explanation. Either remove the example or extend the rule to cover it.
- **Examples after the rule but before the input slot**: structure matters. The model expects rule → examples → real input. Examples after the input get treated as additional input.
- **Adversarial example weights**: if you put the trickiest case first, the model anchors on it. Put a clear case first to teach the format, then move to edge cases.

## Further reading

- [Anthropic — Use examples (multishot prompting)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#use-examples-multishot-prompting) — first-party guidance with balanced-example recommendations.
- [Prompting Guide — Few-Shot Prompting](https://www.promptingguide.ai/techniques/fewshot) — survey of few-shot patterns and failure modes.
- [Brown et al. 2020 — Language Models are Few-Shot Learners](https://arxiv.org/abs/2005.14165) — the original GPT-3 paper on few-shot behavior.
