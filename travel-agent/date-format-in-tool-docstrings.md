# Documenting date formats in tool docstrings

When you expose a tool to an LLM agent, the docstring is the *contract*. If a parameter takes a date string, the docstring must spell out the exact format — `YYYY-MM-DD`, ISO 8601, Unix timestamp, whatever it is. Without the format, the agent will guess, and guessing fails silently.

## Why it matters

LLMs are statistically prone to natural-language date forms. Given a parameter described as `date (str): the date`, an agent will happily pass `"June 10"`, `"10/06/2025"`, `"next Tuesday"`, or `"2025-06-10T00:00:00Z"` — and any of those will return empty results from an API that expects `YYYY-MM-DD`. Worse, the failure is silent: the tool returns `[]`, the agent sees "no activities for that date," and proceeds to build a fictional itinerary.

The cost compounds in a ReAct loop. The agent calls the tool, gets empty results, hallucinates content to fill the gap, runs validation, validation fails on a different axis, the agent retries with a different bad-format date, and the loop burns steps fixing the wrong problem.

## The pattern

```python
def get_activities_by_date_tool(date: str, city: str) -> list[Activity]:
    """Fetch activities available on a specific date in a specific city.

    Use this tool when building an itinerary day. Call once per day per city.

    Args:
        date: The date to fetch activities for, formatted as YYYY-MM-DD
              (ISO 8601 calendar date). Example: "2025-06-10".
        city: The destination city name. Must match a city in the
              activities database. Example: "AgentsVille".

    Returns:
        A list of Activity objects for that date and city. Returns an
        empty list if no activities are scheduled (not an error).
    """
    ...
```

Three things to notice:

1. **Format is named twice**: human-readable ("YYYY-MM-DD") and standards-referenced ("ISO 8601"). The agent picks up on either.
2. **A concrete example string** ("2025-06-10") removes ambiguity that a description alone can't.
3. **Empty-result semantics are documented** — empty list is data, not an error. Without this, agents sometimes retry with format variants when they shouldn't.

## Common pitfalls

- **Vague type hints with no format**: `date (str): the date` — the agent has to invent the format.
- **Format in the example but not in the description**: the agent may read either; both belong.
- **Mismatched casing or separators**: documenting `yyyy-mm-dd` (lowercase) when the API requires `YYYY-MM-DD` won't break anything but suggests imprecision elsewhere.
- **Naming the parameter `date_str` then documenting it as a `datetime`**: name and type and description must agree.
- **Documenting the parameter in a markdown cell above the function instead of in the docstring**: agent-side tool definitions read the function's `__doc__`, not the surrounding notebook narrative.

## Further reading

- [Anthropic — Writing tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents) — broader essay on tool description quality and parameter disambiguation.
- [Anthropic — Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use) — the JSON-schema contract that Python docstrings get translated into for the Anthropic API.
- [PEP 257 — Docstring Conventions](https://peps.python.org/pep-0257/) — Python-side conventions for Args sections that tool-extraction libraries parse.
