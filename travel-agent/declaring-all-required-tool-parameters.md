# Declaring all required tool parameters

Every parameter a function needs at runtime must also appear in the tool's exposed schema. The agent only knows the schema. If the function signature requires `(date, city)` but the schema only declares `date`, the agent will call the tool with one argument and get a `TypeError`.

## Why it matters

A common shortcut during development: write the function with one parameter (`date`), get it working, then add a second (`city`) when filtering becomes necessary — but forget to update the docstring and the agent-side tool definition. The Python function works fine when you call it manually; the agent calls it and the loop breaks.

This is silently expensive in production. A single `TypeError` on a tool call sends the agent into either a retry storm (guessing arg names until something sticks) or an early-exit (returning a partial answer that says "I tried but couldn't fetch the data"). Either way, latency and token cost spike while the underlying issue is a one-line fix.

## The pattern

```python
def get_activities_by_date_tool(date: str, city: str) -> list[Activity]:
    """Fetch activities available on a specific date in a specific city.

    Args:
        date: The date to fetch activities for, formatted as YYYY-MM-DD.
        city: The destination city. Must match a city in the database.

    Returns:
        A list of Activity objects for that date and city.
    """
    return activities_db.query(date=date, city=city)
```

When this tool is registered with the agent runtime, the schema gets generated from the signature + docstring. Both parameters appear. The agent's tool-selection logic now knows both are required, and the LLM gets prompted with the full call signature.

If `city` is left out of either:

- **Missing from the function signature**: the function returns all-city results, the agent has no way to filter, downstream filtering happens in fragile post-processing.
- **Missing from the docstring**: the schema gets generated without it (or with no description), the LLM doesn't know to pass it, gets a `TypeError`, the loop breaks.
- **Missing from the registered tool description string** (separate from the Python docstring, in projects that maintain both): the ReAct prompt's tools section lists `get_activities_by_date_tool(date)`, the LLM emits actions without `city`, runtime fails.

## Common pitfalls

- **Function takes (date, city) but the ReAct prompt lists `get_activities_by_date_tool(date)`**: two places to keep in sync; people remember one.
- **Parameter name drift**: function takes `city`, docstring documents `destination`, ReAct prompt mentions `city_name`. The LLM picks one form and the runtime rejects it.
- **Optional vs required not declared**: in the schema, `city: str = "AgentsVille"` defaults to a city, but if the schema omits the default, the LLM thinks it's required and may refuse to call the tool when no city is in context.
- **Adding the parameter to the function but forgetting to update existing examples in the prompt**: the LLM follows the example, not the signature.

## Further reading

- [OpenAI Cookbook — How to call functions with chat models](https://developers.openai.com/cookbook/examples/how_to_call_functions_with_chat_models) — runnable example showing schema/function alignment for function calling.
- [Anthropic — Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use) — the JSON-schema contract every parameter must appear in.
