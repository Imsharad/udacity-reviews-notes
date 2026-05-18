# Terminating ReAct loops explicitly

A ReAct agent that doesn't know how to stop will either loop until the runtime kills it (max-step error, often after burning real money on tokens) or stop prematurely without producing the structured output the calling code expects. The prompt has to designate exactly one exit and treat every other path as "keep going."

## Why it matters

The THOUGHT → ACTION → OBSERVATION cycle is open-ended by default. The model can emit a new THOUGHT after each OBSERVATION indefinitely. Termination is not implicit — there has to be a specific signal the runner recognizes as "done."

The most common pattern: a `final_answer_tool` (sometimes called `submit_answer`, `done`, or `return_to_user`) that, when called, terminates the loop and returns the tool's argument as the agent's final output. If the prompt doesn't designate this tool as the only exit, three failure modes appear:

1. **The agent loops past completion**: it has the answer, doesn't realize it should stop, calls more tools, hits the max-step cap, returns an error.
2. **The agent stops early without `final_answer_tool`**: it emits a final THOUGHT like "I'm done now" and the loop ends without a structured return value — the calling code gets `None` or a parse error.
3. **The agent calls `final_answer_tool` prematurely**: without a defined gate (see [validation-gates-before-final-answer](./validation-gates-before-final-answer.md)), it exits before validation.

The fix is two complementary rules in the prompt: `final_answer_tool` is the *only* way to exit, and conditions for calling it are explicit.

## The pattern

```python
ITINERARY_REVISION_AGENT_SYSTEM_PROMPT = """
You are a travel-itinerary revision agent.

LOOP TERMINATION:

The ONLY way to end this conversation is to call `final_answer_tool`
with the final TravelPlan as its argument. Calling final_answer_tool
ends the loop; no further THOUGHT/ACTION cycles will execute.

You must call final_answer_tool when:
  - run_evals_tool has returned "FULLY_INCORPORATED" on the current plan, AND
  - all traveler feedback items have been addressed in the plan.

You must NOT exit by:
  - Emitting a THOUGHT like "I'm done" without calling final_answer_tool.
  - Calling any other tool with the intent of finishing.
  - Returning prose instead of a tool call.

If you reach a state where you don't know what to do next, your next
ACTION should be a diagnostic tool call (e.g., run_evals_tool to
check the current plan state), not an exit attempt.
"""
```

The three parts:

1. **The exit is named**: `final_answer_tool`, specifically and exactly. Not "the final tool" or "the answer tool."
2. **The argument shape is implied** (and ideally documented in the tool's own docstring): final_answer_tool takes the structured TravelPlan, not free-form prose.
3. **Negative cases are listed**: ways the agent might try to exit that don't count. This prevents the agent from "creatively" terminating.

## Common pitfalls

- **`final_answer_tool` listed only in the tools section, not as the exit instruction**: the model knows the tool exists but treats it as one option among many.
- **Multiple plausible exits**: prompt mentions both `final_answer_tool` and `submit_plan`; only one is registered with the runtime; the agent guesses wrong.
- **Exit instruction in the middle of a long tool list**: easy to miss. Put termination in its own clearly-marked section, near the top of the prompt.
- **No "you have reached the answer" trigger condition**: the agent doesn't know *when* to call final_answer_tool, so it either calls it too early or never calls it.
- **Conflict with the validation gate**: prompt says "call final_answer_tool when the plan is good" but doesn't define how the agent knows the plan is good. Define this via `run_evals_tool`'s return value, not the agent's self-assessment.

## Further reading

- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — covers termination conditions and stopping rules in agent design.
- [DAIR.AI — ReAct prompting](https://www.promptingguide.ai/techniques/react) — the underlying reasoning-loop structure that termination wraps.
