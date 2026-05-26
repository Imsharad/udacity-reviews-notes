# Exact output labels for classifier prompts

When a model's output gets parsed by exact string match downstream, the labels you ask for must match the labels the parser checks for, byte-for-byte. A weather-compatibility evaluator that returns `"yes"` instead of `"IS_COMPATIBLE"` is functionally broken, even though "yes" is a reasonable answer to the question.

This is the most common silent failure in agent systems that wire LLM outputs into deterministic downstream logic.

## Why it matters

The model can produce any of a hundred phrasings for "this activity is fine in this weather": "yes", "compatible", "no issues", "looks good", "IS_COMPATIBLE", "Compatible.". A downstream parser that does `if response == "IS_COMPATIBLE":` accepts exactly one of those. The other ninety-nine fall through to the else branch and look like the activity was rejected.

In a planning loop, this surfaces as: the agent rejects activities that should be approved, the itinerary is too sparse, the user complains. The bug is one line of prompt text.

The fix is to make the output labels load-bearing in the prompt — not just mentioned, but stated as the only valid outputs.

## The pattern

```python
ACTIVITY_AND_WEATHER_ARE_COMPATIBLE_SYSTEM_PROMPT = """
You are a weather-compatibility evaluator. Given a planned activity
and the weather forecast for that day, decide whether the activity is
safe and enjoyable in that weather.

## Output

Your output must be EXACTLY one of these two strings, with nothing
before or after:

- IS_COMPATIBLE
- IS_INCOMPATIBLE

Do not return JSON. Do not return prose. Do not return "yes", "no",
"compatible", or any other variation. The downstream parser only
recognizes the two strings above.

## Examples

Input: Hike in Yosemite, forecast: clear, 72F.
Output: IS_COMPATIBLE

Input: Outdoor picnic, forecast: thunderstorm, 55F.
Output: IS_INCOMPATIBLE
"""
```

Three things are doing work:

1. **The output section names the labels explicitly**, not just "return whether it's compatible."
2. **The negative space is enumerated**: don't return JSON, don't return "yes" / "no". Models will pick those if you don't rule them out.
3. **Examples lock in the format**: a one-line input maps to a one-line output that is *exactly* one of the two labels.

## Common pitfalls

- **Labels described, not specified**: "Return whether the activity is compatible." The model picks a phrasing, often something polite and verbose.
- **Labels named once, no example**: "Return IS_COMPATIBLE or IS_INCOMPATIBLE." Better, but the model sometimes wraps them in punctuation: `"IS_COMPATIBLE."`, `"**IS_COMPATIBLE**"`, `'"IS_COMPATIBLE"'`. The parser misses all of them.
- **Mixed case**: prompt says "IS_COMPATIBLE", parser checks `.lower() == "is_compatible"`. The case mismatch lives undetected until a model varies its capitalization.
- **Two labels asked, three observed**: the model occasionally invents a third label like `"UNCERTAIN"` or `"PARTIALLY_COMPATIBLE"` when the input is ambiguous. The fix is to instruct the model to default to one of the two labels when uncertain, and pick which one.
- **Labels embedded in a sentence**: "The activity IS_COMPATIBLE with the weather." Parser fails. The output must be the label and nothing else.

## Further reading

- [Anthropic — Be clear and direct](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#be-clear-and-direct) — first-party on specifying exact output format.
- [Anthropic — Use examples (multishot prompting)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#use-examples-multishot-prompting) — covers locking in format via examples.
- [OpenAI Cookbook — Structured outputs](https://developers.openai.com/cookbook/) — for cases where you can move from string-match to structured output mode.
