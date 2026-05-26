# Writing a real evaluate-retrieval tool

The evaluate-retrieval tool is the most commonly stubbed-out component of the UdaPlay agent. It looks small ("did the retrieval return useful results?") and it sits between two more glamorous tools, so submissions often ship with a one-line implementation that always returns `{"sufficient": True}`. The whole fallback architecture collapses.

A real evaluate tool inspects the retrieval results, decides whether they actually answer the query, and returns a structured verdict the agent can act on. Done well, it's the difference between an agent that knows what it doesn't know and one that hallucinates.

## Why it matters

The evaluate step is the routing decision. It's the only thing that distinguishes "always use local" from "always use web" from "use the right one." If it's broken, the agent's behavior is either constantly degraded (always-local on out-of-corpus questions) or constantly wasteful (always-web ignores the local store).

It's also the easiest tool for a reviewer to check. They open the tool's implementation in the notebook, read it in five seconds, and immediately see whether it's a real check or a passthrough.

## The pattern

Two reasonable implementations, depending on how much LLM budget you want to spend.

**Heuristic version** (no extra LLM call):

```python
def evaluate_retrieval_tool(
    query: str,
    retrieved: list[str],
    similarity_scores: list[float] | None = None,
    top_score_threshold: float = 0.65,
) -> dict:
    if not retrieved:
        return {"sufficient": False, "reason": "no results returned from local store"}

    if similarity_scores:
        top_score = max(similarity_scores)
        if top_score < top_score_threshold:
            return {
                "sufficient": False,
                "reason": f"best similarity {top_score:.2f} below threshold {top_score_threshold}",
            }

    # Cheap relevance check: do any of the query's content words appear in
    # the top retrieved document? This catches the "ChromaDB returned 5
    # results that are all about unrelated games" failure mode.
    query_terms = {w.lower() for w in query.split() if len(w) > 3}
    top_doc_terms = set(retrieved[0].lower().split())
    overlap = query_terms & top_doc_terms
    if not overlap and len(query_terms) > 2:
        return {"sufficient": False, "reason": "top document shares no content terms with the query"}

    return {"sufficient": True, "reason": "top result passes threshold and overlap checks"}
```

**LLM-judge version** (one extra call, more accurate):

```python
def evaluate_retrieval_tool(query: str, retrieved: list[str]) -> dict:
    if not retrieved:
        return {"sufficient": False, "reason": "no results returned"}

    judge_prompt = f"""
    You evaluate whether the retrieved documents below contain enough
    information to answer the user's question. Return JSON with two
    keys: "sufficient" (bool) and "reason" (one short sentence).

    User query: {query}

    Top retrieved documents:
    {chr(10).join(f"[{i+1}] {d[:500]}" for i, d in enumerate(retrieved[:3]))}
    """
    response = llm_client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=200,
        messages=[{"role": "user", "content": judge_prompt}],
    )
    return json.loads(response.content[0].text)
```

The LLM-judge version is the right default for UdaPlay — the latency cost is small (haiku-tier model on a short prompt) and the routing decision is more accurate. The heuristic version is what to fall back to if you're rate-limited.

## Common pitfalls

- **`return {"sufficient": True}`**: the placeholder that never got replaced.
- **`return len(retrieved) > 0`**: ChromaDB returns results even on irrelevant queries (up to `n_results`). This evaluate never returns False.
- **No similarity threshold**: the function returns sufficient regardless of how poorly the top result matches. Set a threshold around 0.6-0.7 for cosine distance.
- **Threshold tuned on the training data**: the threshold was set to whatever made the demo queries pass. It doesn't generalize. Pick a threshold by looking at failure cases and adjusting on those, not on the good cases.
- **Evaluate returns a string, not structured JSON**: the agent's calling code does `if evaluation:` which is always truthy for non-empty strings. Use a structured return shape.
- **Evaluate never logs its reasoning**: when it returns False, you don't know why. Always include a `reason` field so the saved notebook output explains the routing decision.

## Further reading

- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — discusses LLM-as-judge for routing decisions inside agents.
- [Anthropic — Increase output consistency](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/increase-consistency) — JSON-shaped outputs from LLM judges.
- [ChromaDB — Querying collections](https://docs.trychroma.com/docs/querying-collections/query-and-get) — the `distances` field that lets you do similarity thresholding without an LLM call.
