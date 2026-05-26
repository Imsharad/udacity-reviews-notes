# Encapsulating the agent as a class

A procedural script with `history = []` at the top and a sequence of `answer_query(...)` calls technically works for the UdaPlay demonstration, but it doesn't model what an agent *is*. An agent has identity, configuration, and state — three things that belong inside an object, not scattered across module-level variables.

Criterion 3 looks for the encapsulation. A class with `__init__` setting up the model client, tool registry, and conversation history; an `answer()` method that takes a query and uses the instance's state; optionally a `reset()` method for starting fresh.

## Why it matters

The class shape forces three things the procedural form usually misses:

1. **State lives in one place** (`self.history`, `self.tools`, `self.client`). The procedural form scatters these across the notebook, making it ambiguous what counts as agent state vs scratch.
2. **The agent is reusable**. Two agents with different configurations (one with web search, one without; one persistent, one ephemeral) can be instantiated as different objects rather than swapped at module level.
3. **The conversation lifecycle is explicit**. `agent.reset()` starts a new conversation. `agent.history` is the audit trail.

The criterion exists because a procedural agent often hides bugs: variables get shadowed, history accidentally gets replaced, the LLM client gets re-initialized per call. The class form makes the lifecycle visible.

## The pattern

```python
class UdaPlayAgent:
    def __init__(
        self,
        llm_client,
        vector_collection,
        web_search_tool,
        system_prompt: str,
        model: str = "claude-sonnet-4-6",
    ):
        self.client = llm_client
        self.collection = vector_collection
        self.web_search = web_search_tool
        self.system_prompt = system_prompt
        self.model = model
        self.history: list[dict] = []

    def answer(self, query: str) -> dict:
        """Answer one query, updating internal state. Returns the answer dict."""
        self.history.append({"role": "user", "content": query})

        # internal -> evaluate -> web fallback
        internal_results = self.collection.query(query_texts=[query], n_results=5)
        evaluation = evaluate_retrieval_tool(query, internal_results["documents"][0])

        if evaluation["sufficient"]:
            source = "internal"
            context = internal_results["documents"][0]
            sources = internal_results["metadatas"][0]
        else:
            web_result = self.web_search(query=query)
            source = "web"
            context = web_result["snippets"]
            sources = web_result["urls"]

        response = self.client.messages.create(
            model=self.model,
            system=self.system_prompt,
            messages=self.history,
            max_tokens=1024,
        )
        answer_text = response.content[0].text
        self.history.append({"role": "assistant", "content": answer_text})

        return {
            "answer": answer_text,
            "route": source,
            "sources": sources,
            "evaluation": evaluation,
        }

    def reset(self):
        """Start a fresh conversation."""
        self.history = []
```

The demonstration in the notebook becomes:

```python
agent = UdaPlayAgent(
    llm_client=anthropic_client,
    vector_collection=collection,
    web_search_tool=tavily_search,
    system_prompt=AGENT_SYSTEM_PROMPT,
)

for query in demo_queries:
    result = agent.answer(query)
    print(f"USER: {query}")
    print(f"AGENT [{result['route']}]: {result['answer']}")
    print(f"SOURCES: {[s.get('title') or s for s in result['sources']]}")
    print(f"EVALUATION: {result['evaluation']}")
    print()
```

The reviewer reads this and immediately sees the class, the instance, the methods, and the per-query state being threaded through.

## Common pitfalls

- **No class at all**: the agent is a top-level function. State is module-level. The criterion will note the missing encapsulation.
- **Class exists but `__init__` is empty / pass**: the class is a namespace, not a stateful object. Methods access module globals instead of `self.*`.
- **History stored as a class attribute, not an instance attribute**: `history = []` at class scope instead of `self.history = []` inside `__init__`. All instances share the same list — looks fine in a single-instance demo, breaks if a second agent is created.
- **Tools passed as kwargs to every method instead of registered in `__init__`**: defeats the encapsulation. The tool registry belongs to the agent instance.
- **No `reset()` method, no way to start fresh**: forces re-instantiation for the next conversation. Acceptable but annoying.
- **The Agent class is defined but never instantiated**: the demonstration still uses the old procedural calls. Reviewer reads the class definition, scrolls down, sees it's unused, fails the criterion.

## Further reading

- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — frames the agent as a structured object with tools, memory, and a control loop.
- [Maintaining state across turns](./maintaining-state-across-turns.md) — the state-preservation problem the class form solves cleanly.
