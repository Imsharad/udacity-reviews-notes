# The internal -> evaluate -> web fallback flow

The UdaPlay agent's signature control flow has three steps in this order:

1. **Internal**: query the local ChromaDB vector store for relevant game records.
2. **Evaluate**: judge whether the internal results are good enough to answer.
3. **Web fallback**: if internal results are insufficient, call the web-search tool.

A submission that skips the evaluate step, or that calls web search on every query, or that never calls web search at all, misses the architectural point of the assignment. The criterion checks for the *flow*, not just for the presence of each tool.

## Why it matters

Without the evaluate step, the agent has no way to know when its local knowledge is incomplete. It will confidently answer "I don't know" on a question about a game released after the dataset was built, even though a web search would have found it in two seconds.

Without the web-search step at all, the agent is purely a RAG system over a static corpus. The "agent" framing is lost — there's no decision-making about which tool to use.

Calling web search on every query (no internal step at all, or always falling through) wastes the local knowledge base and the latency budget. Each query takes 3-5x longer than it should.

The flow is the architecture. Reviewers check for evidence of all three steps in the saved notebook outputs.

## The pattern

```python
def answer_query(query: str, agent_history: list) -> dict:
    # Step 1: internal RAG
    internal_results = collection.query(
        query_texts=[query],
        n_results=5,
    )
    internal_docs = internal_results["documents"][0]
    internal_metas = internal_results["metadatas"][0]

    # Step 2: evaluate
    evaluation = evaluate_retrieval_tool(
        query=query,
        retrieved=internal_docs,
    )
    # evaluation = {"sufficient": bool, "reason": str}

    if evaluation["sufficient"]:
        # Step 3a: answer from internal
        return generate_answer(
            query=query,
            context=internal_docs,
            sources=internal_metas,
            history=agent_history,
        )
    else:
        # Step 3b: web fallback
        web_results = web_search_tool(query=query)
        return generate_answer(
            query=query,
            context=web_results["snippets"],
            sources=web_results["urls"],
            history=agent_history,
            fallback_reason=evaluation["reason"],
        )
```

The reviewer should be able to read the notebook output of three example queries and see:

- Query 1 hits internal, evaluate returns sufficient, agent answers from local.
- Query 2 hits internal, evaluate returns insufficient (e.g., recent release), agent falls back to web.
- Query 3 demonstrates the flow on a fundamentally different topic.

If your three queries all take the same path, the demonstration is incomplete.

## Common pitfalls

- **No evaluate step**: the agent runs internal search and immediately produces an answer, with no check on whether the results were relevant. Often the LLM patches over poor retrieval, hiding the bug.
- **Evaluate step that always returns True**: a stub implementation. Same effect as having no evaluate step at all.
- **Web search called first, internal never used**: easy to write because web search has fewer setup steps. Defeats the whole point of the local corpus.
- **Evaluate step uses simple length heuristic**: `if len(retrieved) > 0: return True`. ChromaDB always returns results unless the collection is empty, so this evaluate always passes. Use a similarity-score threshold or an LLM judge.
- **No fallback demo in the notebook**: all three queries hit the local store. The reviewer can't tell whether the web-search path works at all.
- **The flow not surfaced in output**: the code does the right thing, but the notebook output only shows the final answer. Reviewer needs to see "internal returned X, evaluate said Y, falling back to web" in printed output.

## Further reading

- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — the broader essay on routing patterns and when to add control flow vs let the model decide.
- [Anthropic — Tool use overview](https://platform.claude.com/docs/en/docs/agents-and-tools/tool-use/overview) — defining each step as its own tool that the model can compose.
