# Surfacing reasoning traces in notebooks

An agent's reasoning happens inside the model call. The user (or reviewer) only sees what the notebook prints. If you print only the final answer, the agent could be making decisions in any direction and you'd never know. If you print the reasoning trace too, the demonstration becomes legible.

Criterion 4 asks for evidence of the agent's reasoning. The most common failure is a notebook that prints `final_answer` and nothing else.

## Why it matters

The reasoning trace is the difference between "the agent gave the right answer" and "I can see how the agent arrived at the answer." For a demonstration project, the latter is the whole point. A black-box agent that produces correct outputs is fine for production; a black-box agent in a teaching artifact is unfinished.

Three things should appear in the saved output for each query:

1. **The retrieval step**: what was searched, how many results, what the top result was.
2. **The routing decision**: did the agent use internal knowledge, or fall back to web search? Why?
3. **The final answer**: the user-facing response, ideally with inline citations.

If steps 1 and 2 are missing, the reviewer sees only step 3 and can't grade the agent's process.

## The pattern

Add explicit logging to the agent's main entry point. In a notebook, `print()` is usually enough; for a real product you'd use structured logging.

```python
def answer(self, query: str) -> dict:
    print(f"\n=== Query: {query} ===")
    self.history.append({"role": "user", "content": query})

    # Step 1: retrieval
    internal = self.collection.query(query_texts=[query], n_results=5)
    docs, scores = internal["documents"][0], internal.get("distances", [[]])[0]
    print(f"[retrieval] {len(docs)} results, top score: {scores[0]:.3f}" if scores else f"[retrieval] {len(docs)} results")
    if docs:
        top_title = internal["metadatas"][0][0].get("title", "(no title)")
        print(f"[retrieval] top result: {top_title}")

    # Step 2: evaluation
    evaluation = evaluate_retrieval_tool(query, docs)
    print(f"[evaluate] sufficient={evaluation['sufficient']}  reason: {evaluation['reason']}")

    # Step 3: route
    if evaluation["sufficient"]:
        print("[route] using internal knowledge")
        context, sources = docs, internal["metadatas"][0]
    else:
        print("[route] falling back to web search")
        web = self.web_search(query=query)
        context, sources = web["snippets"], web["urls"]
        print(f"[route] web returned {len(web['snippets'])} snippets")

    # Step 4: synthesis
    response = self.client.messages.create(
        model=self.model,
        system=self.system_prompt,
        messages=self.history + [{"role": "user", "content": render_with_context(query, context)}],
        max_tokens=1024,
    )
    answer = response.content[0].text
    self.history.append({"role": "assistant", "content": answer})

    print(f"[answer] {answer}")
    print(f"[sources] {[s.get('title') or s for s in sources]}")

    return {"answer": answer, "route": evaluation, "sources": sources}
```

After running three demo queries, the notebook output reads like a trace log. The reviewer can scroll through it and see exactly what the agent did at each step.

## Common pitfalls

- **Only the final answer printed**: the reviewer sees the result, not the process. No way to grade the reasoning.
- **Logging via `logging.info()`**: cells that don't configure logging output produce nothing visible. In a notebook, prefer `print()` so the output goes to the cell output directly.
- **Logging too verbose**: every embedding similarity score, every tool argument, every token of the response. Output is unreadable. Pick the load-bearing decisions and log those.
- **Logging only in code, not in saved output**: the `print()` calls exist but the cell was run before they were added. Re-run the cell so the new output is saved.
- **Reasoning produced inside the LLM but discarded**: if you use Anthropic's extended thinking, the thinking blocks should also be surfaced in the printed output, not stripped out.
- **One log line, multi-step trace inside it**: `print(f"agent did: {result}")` where `result` is a dense dict. Hard to scan. Break the log into one line per decision.

## Further reading

- [Anthropic — Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking) — when to surface the model's own thinking trace alongside the answer.
- [Showing tool invocations in output](./showing-tool-invocations-in-output.md) — the sibling pattern for tool-call visibility specifically.
