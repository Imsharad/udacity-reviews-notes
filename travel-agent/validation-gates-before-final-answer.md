# Validation gates before the final answer

If your agent has a `final_answer_tool`, it should not be reachable until a `run_evals_tool` (or equivalent validator) has returned a passing result. "The agent should validate before finalizing" is not enough as a prompt instruction — it has to be a hard sequencing rule the agent can't bypass.

## Why it matters

When the validation step is suggested rather than required, the agent skips it more often than you'd expect. Especially under token pressure, after a long sequence of tool calls, or when the agent has just produced what looks like a good answer — it jumps straight to `final_answer_tool` and the user receives a plan that still has hidden budget overruns, weather conflicts, or unaddressed feedback.

This is a "looks correct, isn't" failure mode, which is the worst kind in agent systems. The user sees a complete-looking itinerary, doesn't run it through their own validation, and finds out the budget is double the cap only after they've already committed to it.

The fix is structural, not exhortative. The prompt needs to define the sequencing as a contract: `final_answer_tool` is only valid *after* the most recent `run_evals_tool` call returned a passing result.

## The pattern

```python
ITINERARY_REVISION_AGENT_SYSTEM_PROMPT = """
You are a travel-itinerary revision agent.

EXIT CONDITIONS (mandatory sequencing):

You may only call `final_answer_tool` immediately after a successful
`run_evals_tool` call on the current revised plan. "Successful" means
run_evals_tool returned status == "FULLY_INCORPORATED".

If run_evals_tool returns any other status (e.g., "BUDGET_EXCEEDED",
"WEATHER_CONFLICT", "FEEDBACK_NOT_INCORPORATED"), you must revise
the plan, call run_evals_tool again on the revised plan, and only
then proceed toward final_answer_tool.

You may NOT:
- Call final_answer_tool without an immediately-preceding run_evals_tool call.
- Call final_answer_tool with a plan that differs from the one most recently
  passed to run_evals_tool.
- Skip validation because the plan "looks fine" — only run_evals_tool's
  return value determines if it's fine.

Available tools (in typical sequencing order):
  1. get_activities_by_date_tool — fetch options
  2. calculator_tool — check budget math
  3. run_evals_tool — validate the candidate plan
  4. final_answer_tool — return the plan (ONLY after #3 returns passing)
"""
```

Three things make this gate hold:

1. **Exact tool name** (`run_evals_tool`) — not a generic phrase like "validation tools." The LLM matches on names; "validation tools" is ambiguous.
2. **Exact passing condition** (`"FULLY_INCORPORATED"`) — the prompt names the string the agent must see, removing interpretation.
3. **Explicit negative cases** — listing what the agent may NOT do is more effective than only listing what it should do. The LLM has a clear contradiction signal if it considers a bypass.

## Common pitfalls

- **"Run evaluation tools before finalizing"** — vague, easily skipped. The LLM treats it as advisory.
- **Validator named generically in prompt, exactly in code** — the prompt says "run the evals," the runtime expects `run_evals_tool`; the LLM emits `run_evaluation` or `evaluate` and the runner errors.
- **Validation passing condition not specified** — the agent runs `run_evals_tool`, gets back `{"status": "FEEDBACK_NOT_INCORPORATED", ...}`, and treats any non-error response as success.
- **Final-answer gate enforced only at code level, not in prompt** — the code might reject premature `final_answer_tool` calls, but the agent has no way to know that and burns steps emitting them.

## Common debugging signal

If the agent's saved trace shows `final_answer_tool` being called without a preceding `run_evals_tool` call, the gate isn't in the prompt — or it's in the prompt but worded weakly enough that the model is ignoring it.

## Further reading

- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — covers evaluator-optimizer loops and gated termination patterns (note: this essay covers the principle but doesn't show the exact final-answer gate; the gate is a project-specific refinement).
