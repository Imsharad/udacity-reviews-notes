# Three or more demo queries

UdaPlay criterion 4 asks the agent to be demonstrated on at least three distinct example queries. Many submissions show one. Sometimes two. Occasionally a single query with the answer printed three times. The agent might work perfectly; without enough demonstration coverage, the criterion fails.

This is the most mechanical of the demonstration checks. The fix is one for-loop and a list of queries.

## Why it matters

A single demo query proves the agent runs. Three queries prove the agent's behavior generalizes — that the same code handles different inputs, different routing decisions, and different output shapes. The criterion is designed to catch agents that work only for the one query the developer happened to test.

Three is the minimum, not the target. Five well-chosen queries are better than three repeated variants.

## The pattern

```python
demo_queries = [
    # In-corpus, single-hop: should use internal retrieval
    "What year was The Legend of Zelda: Ocarina of Time released?",

    # Out-of-corpus or recent: should trigger web fallback
    "What did critics say about Tears of the Kingdom?",

    # Multi-turn follow-up to demonstrate state preservation
    # (run after the prior query; relies on its context)
    "Who developed it?",
]

agent = UdaPlayAgent(...)

for i, query in enumerate(demo_queries, start=1):
    print(f"\n{'=' * 60}\nQuery {i}: {query}\n{'=' * 60}")
    result = agent.answer(query)
    print(f"\nFinal answer: {result['answer']}")
    print(f"Route: {result['route']}")
    print(f"Sources: {result['sources']}")
```

Three queries, each chosen to exercise a different part of the agent: in-corpus retrieval, web fallback, and multi-turn state. The notebook now has three populated cells of demonstration evidence.

## How to pick three queries

Use this as a checklist; each demo query should hit at least one row:

| Coverage axis | Why include it |
|---|---|
| **In-corpus, simple** | Proves retrieval works at all |
| **In-corpus, multi-document** | Proves retrieval returns multiple games when the question warrants it |
| **Out-of-corpus / recent release** | Proves web fallback fires |
| **Follow-up using prior context** | Proves multi-turn state works |
| **Ambiguous query** | Proves evaluate-retrieval routes correctly under uncertainty |

Three queries can cover three of these axes. Five queries can cover all five and leave a margin for any one of them to misfire without tanking the demonstration.

## Common pitfalls

- **One query only**: most common failure for this criterion. Reviewer can't grade generalization.
- **Three queries on the same game**: technically three queries, but they exercise the same code path. Reviewer flags as insufficient.
- **Three queries that all hit internal retrieval**: no demonstration of the web-fallback path. Pair with [diverse-queries-and-fallback-demo](./diverse-queries-and-fallback-demo.md).
- **Three queries hardcoded in markdown**: the queries are listed in a markdown cell but never actually executed. No output, no demonstration.
- **Three queries executed but only the final summary printed**: the loop runs but only the last result is visible in saved output. Print each iteration.
- **Three queries with errors silently swallowed**: try/except wraps the loop, errors are caught, the demonstration looks clean but is empty. Don't catch errors during the demo.

## Further reading

- [Diverse queries and fallback demo](./diverse-queries-and-fallback-demo.md) — sibling page on query selection for coverage.
- [Demonstrating follow-up questions](./demonstrating-followup-questions.md) — for the multi-turn axis.
