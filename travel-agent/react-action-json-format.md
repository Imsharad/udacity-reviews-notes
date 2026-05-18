# Action JSON format in ReAct prompts

In a ReAct loop the agent's `ACTION:` line gets parsed by a runner that calls the tool with the emitted arguments. The runner needs a structured format — typically `{"tool_name": "...", "arguments": {...}}`. If the prompt doesn't specify the format, the LLM emits free-form prose, and the runner can't extract a tool call.

## Why it matters

The ReAct paper's original examples use plain-text labels: `Action 1: Search[Eiffel Tower]`. That syntax was fine for the paper's purpose (illustrating the reasoning pattern), but production runners almost always parse JSON-structured actions. The model has to know which form to emit.

Without an explicit format in the prompt, the LLM falls back on whatever it saw most often during training — which is sometimes the prose form, sometimes a JSON form, sometimes a markdown code block, sometimes natural language ("I'll search for the Eiffel Tower"). The runner sees the variance and fails to parse on the variants it didn't expect.

The failure mode is usually a single malformed action that stops the loop early. The agent emits something the runner can't parse, the runner returns an error to the agent (or worse, silently logs and continues), the agent doesn't get the observation it expected, and the THOUGHT → ACTION → OBSERVATION cycle breaks.

## The pattern

Show the format explicitly in the prompt. Two parts: the format spec, and a concrete worked example.

```python
ITINERARY_REVISION_AGENT_SYSTEM_PROMPT = """
You are a travel-itinerary revision agent. You revise itineraries
based on traveler feedback, then validate the result before returning.

You operate in a ReAct loop with three steps per turn:

  THOUGHT: <your reasoning about what to do next>
  ACTION: {"tool_name": "<tool>", "arguments": {<args>}}
  OBSERVATION: <returned by the tool runner — do not write this yourself>

The ACTION line MUST be valid JSON with exactly two keys: `tool_name`
(string) and `arguments` (object). No prose, no markdown, no comments.

Example turn:

  THOUGHT: The traveler wants to add a museum visit on June 10.
  I need to check what museums are available in AgentsVille that day.
  ACTION: {"tool_name": "get_activities_by_date_tool",
           "arguments": {"date": "2025-06-10", "city": "AgentsVille"}}
  OBSERVATION: [list of activities returned by runner]

Available tools:
  - get_activities_by_date_tool(date: str, city: str) -> list[Activity]
  - calculator_tool(expression: str) -> float
  - run_evals_tool(plan: TravelPlan) -> EvalResult
  - final_answer_tool(plan: TravelPlan) -> None
"""
```

Three load-bearing elements:

1. **The exact JSON shape**, with the two required keys named.
2. **A worked example** showing the format in context — the LLM imitates examples more reliably than it follows abstract specs.
3. **Explicit prohibition** of variants ("No prose, no markdown, no comments") — without this, the LLM will sometimes wrap the JSON in a ```json fenced block, which the runner may or may not strip.

## Common pitfalls

- **Format defined but not exemplified**: the spec says JSON, no example shown, the LLM guesses at the field names (`tool` vs `tool_name`, `args` vs `arguments`).
- **Prompt uses the original ReAct prose form**: `ACTION: search("Eiffel Tower")` — works in the paper, breaks in any production runner that expects JSON.
- **Examples wrapped in markdown code fences**: the LLM imitates and wraps every action in fences; the runner extracts the JSON correctly most of the time but fails on edge cases.
- **OBSERVATION described as something the model writes**: the model "predicts" what the observation will be instead of waiting for the runner to provide it; cycle collapses into a one-shot generation.

## Further reading

- [Anthropic — Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use) — the tool_use JSON contract that ACTION lines need to mirror.
- [DAIR.AI — ReAct prompting](https://www.promptingguide.ai/techniques/react) — the original loop structure (note that the page uses plain-text actions; the JSON form is a production refinement).
