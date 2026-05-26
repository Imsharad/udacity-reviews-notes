# Citing sources in agent answers

An agent that answers "The Legend of Zelda was released in 1986" without saying *where it got that information* is not auditable. A user can't tell whether the answer came from the local game database, from a web search, or from the model's pretraining (which might be wrong). Citations make the trust boundary visible.

For UdaPlay specifically, criteria 3 and 4 both grade the agent on whether final answers include source attribution. The most common failure is an agent that retrieves sources correctly but discards them before generating the answer.

## Why it matters

Citations do three things at once:

1. **Provenance**. The user can verify the answer against the source.
2. **Routing signal**. The user can see whether the agent used local knowledge or fell back to web search.
3. **Reviewer evidence**. The criterion explicitly asks for cited answers. No citations = no points.

A model that has all three retrieved documents in its context but answers without citing any of them isn't broken — it's just not been told that citations are required output. The fix is in the prompt and the post-processing, not in the retrieval.

## The pattern

Two complementary mechanisms — both should be in place.

**1. Prompt-side: instruct the model to cite.**

```python
AGENT_SYSTEM_PROMPT = """
You are a video-game research assistant. You answer questions about
games using the retrieved context provided to you. Your answers must
cite the source for every factual claim.

## Source citation format

When a fact comes from a retrieved game record, end the sentence
with [source_id] where source_id is the id of the game record.

When a fact comes from a web search result, end the sentence with
[web: URL].

If you cannot find a source, say so explicitly: "I don't have a source
for this." Never invent citations.

## Example

Internal source [ocarina_of_time]: The Legend of Zelda: Ocarina of Time
was released in 1998 for the Nintendo 64 [ocarina_of_time]. The game's
director was Eiji Aonuma [ocarina_of_time].

Web source [web: https://...]: Tears of the Kingdom released in May 2023
[web: https://www.nintendo.com/...].
"""
```

**2. Code-side: pass the sources into the prompt and parse them back.**

```python
def generate_answer(query: str, context: list[str], sources: list[dict], history: list) -> dict:
    # Render the retrieved context with stable source ids the model can cite
    rendered_context = "\n\n".join(
        f"[{src['source_file'].replace('.json', '')}]\n{doc}"
        for doc, src in zip(context, sources)
    )

    user_message = (
        f"Question: {query}\n\n"
        f"Retrieved context:\n{rendered_context}"
    )

    response = llm_client.messages.create(
        model="claude-sonnet-4-6",
        system=AGENT_SYSTEM_PROMPT,
        messages=history + [{"role": "user", "content": user_message}],
        max_tokens=1024,
    )
    answer_text = response.content[0].text

    # Extract cited source ids back out so we can show them in the UI / output
    cited_ids = re.findall(r"\[([a-z0-9_]+)\]", answer_text)
    cited_sources = [s for s in sources if s["source_file"].replace(".json", "") in cited_ids]

    return {"answer": answer_text, "sources": cited_sources}
```

Now the saved notebook output for each query shows the answer with inline `[source_id]` tags AND a structured `sources` list that the reviewer can audit against the retrieved chunks.

## Common pitfalls

- **Sources retrieved but not passed into the prompt**: the model has nothing to cite even when it wants to. The retrieval is wasted.
- **Sources passed but no instruction to cite**: the model produces fluent answers without inline tags. The prompt has to ask explicitly.
- **Citation format inconsistent**: prompt says `[source_id]`, examples show `(source_id)`, code parses with `<source_id>`. Three different forms, none match.
- **Generic citations**: "[source]" on every sentence, with no specific id. Useless for audit.
- **Hallucinated citations**: the model invents `[ocarina_of_time_2]` when no such id exists. Defends against by passing the model a closed list of valid source ids and instructing "use only these ids."
- **Citations only on web-search results, not internal**: the project rubric checks both paths. The agent must cite local sources too.
- **Citations only in the final answer object, not in the printed notebook output**: the reviewer reads the notebook and doesn't see them. Print them.

## Further reading

- [Anthropic — Citations](https://platform.claude.com/docs/en/build-with-claude/citations) — first-party support for cited responses with verifiable source spans.
- [Anthropic — Reduce hallucinations](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/reduce-hallucinations) — the broader pattern of grounding answers in retrieved context and refusing when no source exists.
