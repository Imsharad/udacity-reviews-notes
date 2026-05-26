# Maintaining state across turns

A stateful agent is one where turn N can see what happened in turns 1 to N-1. The user can say "what about its sequel?" and the agent knows what "it" refers to. A stateless agent treats each turn as a fresh conversation; pronouns dangle, follow-ups break.

UdaPlay's criterion 3 grades the agent on whether it actually preserves state, and the most common failure is a one-line oversight: the agent's message history list is reset to `[]` at the top of every query function.

## Why it matters

State is what makes an agent feel like an agent. Without it, the system is a sequence of independent question-answer pairs, no different from a search box. With it, the agent supports follow-up questions, topic drilldowns, and corrections — the interactions that prove agency is doing useful work.

For UdaPlay specifically, criterion 3 explicitly requires demonstrating multi-turn state. The notebook must show at least one second-turn query that depends on the first turn's context (e.g., "tell me about Zelda" followed by "what year was it released?" without the agent asking "which Zelda?").

## The pattern

Two reasonable shapes — a function with an explicit history argument, or an Agent class that owns the history.

**Function with history argument:**

```python
def answer_query(query: str, history: list[dict]) -> tuple[str, list[dict]]:
    """Return (answer, updated_history)."""
    history = history + [{"role": "user", "content": query}]

    # Run the internal -> evaluate -> web flow with full history
    context = retrieve_context(query, history=history)
    response = llm_client.messages.create(
        model="claude-sonnet-4-6",
        system=AGENT_SYSTEM_PROMPT,
        messages=history,
        max_tokens=1024,
    )
    answer = response.content[0].text

    history = history + [{"role": "assistant", "content": answer}]
    return answer, history


# Usage in the notebook
history = []
answer1, history = answer_query("Tell me about The Legend of Zelda.", history)
print(answer1)

answer2, history = answer_query("What year was it released?", history)
print(answer2)
```

The two `history = history + [...]` lines are what makes this stateful. The second `answer_query` call sees the user's first question AND the agent's first answer, so "it" resolves to Zelda.

**Agent class** (covered separately in [encapsulating-the-agent-as-a-class](./encapsulating-the-agent-as-a-class.md)) is cleaner once you have more than two pieces of state to track.

## Common pitfalls

- **`history = []` inside the function**: the most common bug. The function looks stateful but resets state every call.
- **History tracked but not passed to the LLM**: the variable accumulates messages but the `messages=` argument to the LLM call is still `[{"role": "user", "content": query}]`. The model never sees the prior turns.
- **History stored on global variable**: technically works for a notebook demo but breaks if multiple "users" or threads exist. Acceptable for the assignment, not for anything beyond it.
- **System prompt re-injected as a user message every turn**: doubles the system message in the history; the model gets confused about role. The `system=` kwarg is separate from the messages list.
- **Tool call results not appended to history**: the agent makes a tool call, gets results, uses them to answer — but the history skips the tool exchange. The next turn loses context about what the agent already looked up.
- **Trimming history too aggressively**: keeping only the last 2 messages to save tokens. Loses any state more than one turn back. If you need to trim, keep at least a few turns or summarize older turns.

## Further reading

- [Anthropic — Multi-turn conversations](https://platform.claude.com/docs/en/build-with-claude/conversations) — how the `messages` array represents an ongoing conversation.
- [Anthropic — Tool use across multiple turns](https://platform.claude.com/docs/en/docs/agents-and-tools/tool-use/implement-tool-use#multi-turn-conversations) — the canonical pattern for including tool-result blocks in conversation history.
