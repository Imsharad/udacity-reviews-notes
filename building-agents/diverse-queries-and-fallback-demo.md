# Diverse queries and the fallback demo

Three queries about three games released the same year, all answerable from the local corpus, do not demonstrate an agent. They demonstrate that retrieval works on easy cases. The criterion specifically asks for evidence that the agent's routing decision is exercised — at least one query must trigger the web-search fallback, and the queries together should cover meaningfully different topics.

## Why it matters

The agent's distinctive behavior is the routing: when to use internal knowledge, when to fall back to the web. If every demo query lands in the same path, that routing is never tested in the saved output. The reviewer can't tell whether the fallback path even works.

Diversity is also a hedge against retrieval bias. Three queries about Nintendo games might all return correct answers; the same agent might fail on a SEGA query because the corpus is Nintendo-heavy. The demonstration should expose those edges.

## The pattern

Pick demo queries that together cover three axes: time, topic, and routing.

**Time axis** — queries about games from different eras:

- A classic (1990s) — should be in-corpus.
- A 2010s release — should be in-corpus.
- A 2024 or later release — should trigger web fallback (corpus is static).

**Topic axis** — queries about different game properties:

- A factual lookup (release year, publisher).
- A descriptive question (gameplay style, setting).
- A comparison or recommendation.

**Routing axis** — at minimum one of each:

- Internal-only (clear in-corpus answer).
- Web fallback (out-of-corpus or recent).
- Multi-turn (follow-up depending on prior context).

A demo set that hits all three axes simultaneously could be:

```python
demo_queries = [
    # In-corpus, classic, factual
    "When was Super Mario 64 released and who developed it?",

    # Web fallback (recent), descriptive
    "How was the launch reception of the 2024 Nintendo Switch 2?",

    # Multi-turn follow-up, in-corpus
    "Going back to Super Mario 64 — what platforms did its sequels release on?",
]
```

Each query exercises a different combination of the agent's capabilities. The saved output of all three together is a complete demonstration.

## What "the fallback fires" looks like in the notebook

The trace for the web-fallback query should make the routing decision visible:

```text
=== Query: How was the launch reception of the 2024 Nintendo Switch 2? ===
[retrieval] 5 results, top score: 0.41
[retrieval] top result: Nintendo Switch
[evaluate] sufficient=False  reason: top similarity 0.41 below threshold 0.65
[route] falling back to web search
[tool] web_search(query='2024 Nintendo Switch 2 launch reception') -> 8 web snippets
[answer] The Nintendo Switch 2 launched in June 2025 [web: https://...]...
```

The reviewer reads this and immediately sees: internal had a near-miss, evaluate flagged it, web fallback ran, answer cites a web URL. Full routing loop, one query.

## Common pitfalls

- **All queries in-corpus**: the web-search tool is never invoked in the saved output. Reviewer can't grade routing.
- **All queries the same topic**: three queries about Mario games. Doesn't prove the agent generalizes.
- **Recent-release query that the corpus secretly answers**: the corpus might include the game you thought was out-of-corpus. Pick something you're certain is post-dataset (or check the corpus before relying on the fallback).
- **Fallback query selected to fail**: the web search returns nothing useful. The fallback fires but the answer is empty. Pick a fallback query the web can actually answer.
- **Demo queries all named in a list but only executed one at a time, with the kernel restart in between**: state isn't preserved across the demonstrations. Multi-turn coverage breaks.
- **The "diverse" set is three paraphrases of the same question**: "When did Mario 64 come out?", "What year was Mario 64 released?", "Mario 64's release date?". Reviewer rightly counts as one query, not three.

## Further reading

- [Three or more demo queries](./three-or-more-demo-queries.md) — the count-side companion (covers "how many"; this page covers "how varied").
- [Internal -> evaluate -> web fallback flow](./internal-evaluate-web-search-fallback-flow.md) — the routing architecture the diverse queries exercise.
