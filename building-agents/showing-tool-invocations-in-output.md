# Showing tool invocations in output

The agent has three tools: internal retrieval, evaluate-retrieval, and web search. Reviewers grading criterion 4 want to see *which* tools fired on each demo query, with *what arguments*, and *what they returned*. A notebook that hides tool calls inside the agent's internals and prints only the synthesized final answer fails this check.

This is a close cousin of the reasoning-trace pattern, but specific to tool calls. Tool invocations are the agent's interaction with the outside world; they deserve their own visibility.

## Why it matters

Tool calls are where the agent's behavior diverges from a plain LLM. The synthesized answer at the end might look the same whether the agent used internal retrieval or web search, but the tool-call sequence is what proves the agent is *acting*, not just *responding*. The criterion exists because reviewers need to verify this for each demo query.

It also helps you debug. An agent that produces wrong answers is much easier to fix when you can see exactly which tool returned what.

## The pattern

Wrap each tool with a thin logging shim, or log explicitly at the call site. The notebook output should answer three questions for each tool call:

1. **Which tool** was invoked (by name).
2. **What arguments** were passed.
3. **What it returned** (summarized — not the full payload).

```python
def log_tool_call(tool_name: str, args: dict, result):
    """Print a one-line trace of a tool invocation."""
    args_str = ", ".join(f"{k}={truncate(v)}" for k, v in args.items())
    result_summary = summarize_result(result)
    print(f"[tool] {tool_name}({args_str}) -> {result_summary}")


def truncate(v, n=60):
    s = str(v)
    return s if len(s) <= n else s[:n] + "..."


def summarize_result(r):
    if isinstance(r, dict):
        if "documents" in r:
            return f"{len(r['documents'][0])} docs"
        if "snippets" in r:
            return f"{len(r['snippets'])} web snippets"
        if "sufficient" in r:
            return f"sufficient={r['sufficient']}"
    return truncate(repr(r))
```

Then in the agent:

```python
def answer(self, query: str) -> dict:
    # internal retrieval
    args = {"query_texts": [query], "n_results": 5}
    internal = self.collection.query(**args)
    log_tool_call("chroma.query", args, internal)

    # evaluation
    args = {"query": query, "retrieved": internal["documents"][0]}
    evaluation = evaluate_retrieval_tool(**args)
    log_tool_call("evaluate_retrieval", args, evaluation)

    if evaluation["sufficient"]:
        context = internal["documents"][0]
    else:
        args = {"query": query}
        web = self.web_search(**args)
        log_tool_call("web_search", args, web)
        context = web["snippets"]

    # ... synthesis ...
```

A demo query's saved output now contains three or four `[tool] ...` lines that map the agent's tool sequence end-to-end. The reviewer reads them top-to-bottom and can grade the criterion in five seconds.

## Common pitfalls

- **Tool calls invisible**: the agent works but the notebook only shows the final answer. No tool evidence.
- **Tool calls logged at INFO level but logging not configured**: the calls happen, the logger swallows the output. Use `print()` in notebook demos.
- **Tool arguments printed but result not summarized**: `[tool] web_search(query='zelda') -> {very long dict}` makes the output unreadable. Summarize.
- **Tool result printed in full**: ChromaDB returns hundreds of lines of nested dicts. Print the count and the top item, not the whole thing.
- **Every internal helper function logged as a tool call**: the trace is too noisy. Only the *agentic* tools (the ones the model chooses to invoke) need this treatment. Internal utility functions don't.
- **Tool calls logged inside the wrapper but the wrapper is not used everywhere**: half the calls go through the logging shim, half are direct. Inconsistent trace. Pick one path and use it consistently.

## Further reading

- [Anthropic — Tool use overview](https://platform.claude.com/docs/en/docs/agents-and-tools/tool-use/overview) — the canonical tool-use loop with tool_use and tool_result content blocks.
- [Anthropic — Tool result formats](https://platform.claude.com/docs/en/docs/agents-and-tools/tool-use/implement-tool-use#handling-tool-results) — how the model sees tool results (useful for understanding what to surface in your trace).
- [Surfacing reasoning traces in notebooks](./surfacing-reasoning-traces-in-notebooks.md) — broader sibling pattern for visibility.
