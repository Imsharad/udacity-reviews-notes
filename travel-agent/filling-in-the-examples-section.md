# Filling in the examples section

A prompt with `## Examples\n\n*********` or `## Examples\n\n[your examples here]` is a prompt with no examples. The model treats the placeholder as either noise or, worse, as a literal example to imitate. Either way, the few-shot scaffolding does nothing.

This is a near-cousin of the unfilled f-string placeholder mistake, but it happens to the *content* rather than the *interpolation*. The notebook ships with an examples block as a TODO marker; the student is meant to fill it in; sometimes they don't.

## Why it matters

The examples section is doing two jobs at once: teaching the output format and teaching the decision boundary. An empty section trains neither. The classifier prompt without examples falls back to whatever the model has seen in pretraining about similar tasks — which is generally fine for casual phrasing but bad for an exact-match output contract.

For the weather-compatibility evaluator specifically, an empty examples block produces outputs like "Yes, this looks compatible." Polite, fluent, completely unparseable.

## The pattern

A filled examples section for a binary classifier:

```python
ACTIVITY_AND_WEATHER_ARE_COMPATIBLE_SYSTEM_PROMPT = """
You are a weather-compatibility evaluator.

Output exactly one of: IS_COMPATIBLE, IS_INCOMPATIBLE.

## Examples

Input: Activity is "outdoor hike", weather is clear and 72F.
Output: IS_COMPATIBLE

Input: Activity is "outdoor picnic", weather is thunderstorm.
Output: IS_INCOMPATIBLE

Input: Activity is "museum tour", weather is heavy rain.
Output: IS_COMPATIBLE

Input: Activity is "beach day", weather is 50F with high winds.
Output: IS_INCOMPATIBLE
"""
```

Each example is one input line and one output line. Same shape every time. The model copies the shape.

## Checklist before shipping

- Replace every `*****`, `<...>`, `[fill in]`, or `TODO` marker in the prompt.
- Read the assembled prompt with `print(PROMPT)` and scan for placeholder leftovers.
- Verify each example produces the exact output string you've specified (no trailing punctuation, no markdown bold, no quotes).
- Check that the examples cover both classes for a binary classifier, multiple classes for a multi-class classifier.
- Make sure no example contradicts your written rule.

## Common pitfalls

- **Examples block exists but empty**: `## Examples` followed by blank lines. Model parses the heading as an instruction it doesn't understand.
- **Examples block contains the literal placeholder text**: the model sometimes echoes the placeholder back as if it were the answer.
- **Examples copied from a different task**: leftover code-review or research examples in a travel-planning prompt. Model gets confused about what task it's doing.
- **Examples that are too easy**: every input is a clear case. Model doesn't learn the boundary. Include one borderline case per class.
- **Examples that are too narrative**: each example is a paragraph instead of one input line + one output line. Model copies the paragraph format and produces verbose outputs.

## Further reading

- [Anthropic — Use examples (multishot prompting)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#use-examples-multishot-prompting) — first-party guidance on filling the examples block well.
- [Anthropic — Be clear and direct](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#be-clear-and-direct) — companion section on the prompt structure that wraps the examples block.
