# Demonstrating follow-up questions

A stateful agent is only stateful if you can see it being stateful. A notebook that shows three independent question-answer pairs proves nothing about state. A notebook that shows a first turn establishing a topic and a second turn referencing it via pronoun or shorthand proves the agent is preserving context.

The criterion checks for the demonstration, not just the capability.

## Why it matters

State preservation is invisible inside the code. The reviewer can't tell from looking at `history.append(...)` whether the agent actually uses that history downstream. The proof has to live in saved notebook output.

The cheapest proof is a multi-turn exchange where turn 2 only makes sense if turn 1 is in context. "What year did it release?" only resolves to "Tears of the Kingdom" if the prior turn was about Tears of the Kingdom. The reviewer reads the exchange and confirms state.

## The pattern

Pick at least one of the demonstration queries to be a follow-up. The exchange should look like this in the notebook:

```python
history = []

# Turn 1: establish a topic
print("USER:", q1 := "Tell me about The Legend of Zelda: Tears of the Kingdom.")
answer1, history = agent.answer(q1, history)
print("AGENT:", answer1)
print()

# Turn 2: reference the topic implicitly
print("USER:", q2 := "What year was it released?")
answer2, history = agent.answer(q2, history)
print("AGENT:", answer2)
print()

# Turn 3: drill deeper on the same topic
print("USER:", q3 := "Was its predecessor on the Switch too?")
answer3, history = agent.answer(q3, history)
print("AGENT:", answer3)
```

Three things make this a real demonstration:

1. Turn 2's question contains "it" — the pronoun forces the agent to use prior context.
2. Turn 3 introduces "its predecessor" — requires the agent to know what game and what platform.
3. The same `history` variable is threaded through all three calls.

If the reviewer can swap the order of the three turns without changing any answers, you haven't demonstrated state.

## Good follow-up patterns to use

- **Pronoun reference**: "what year was *it* released?", "who developed *that*?"
- **Comparison to the prior subject**: "how does that compare to *the original*?"
- **Drilling into a detail**: "tell me more about the developer", after the agent named Nintendo EPD.
- **Asking for sources cited in turn 1**: "where did you find that?" — bonus, demonstrates citation tracking too.

## Common pitfalls

- **All three demo queries independent**: the agent might be stateless and you'd never know. Add at least one follow-up.
- **Follow-up that re-states the topic**: "what year was The Legend of Zelda: Tears of the Kingdom released?" — does not test state because the topic is in the question. Use a pronoun.
- **Follow-up that the agent gets right by accident**: a question generic enough that the model's pretraining covers it. Use a project-specific drill-down ("Was *its predecessor* on the Switch too?") that requires the agent to know what was discussed.
- **History passed to the agent but not surfaced in the output**: the reviewer can't tell from the printed exchange whether turn 2 actually used turn 1. Print the answers with enough detail that the state-use is visible (e.g., the answer to "what year?" includes the title, even though the user didn't restate it).
- **No follow-up in the demonstration, only in the source code**: an `answer_followup_question` function exists but no example call appears in the notebook output. No evidence.

## Further reading

- [Anthropic — Multi-turn conversations](https://platform.claude.com/docs/en/build-with-claude/conversations) — first-party patterns for follow-up handling.
- [Maintaining state across turns](./maintaining-state-across-turns.md) — the implementation side of the same problem (this page covers the demonstration side).
