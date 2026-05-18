# Travel Agent — Engineering Notes

Concept notes for the AgentsVille trip-planner project: a travel agent that builds a day-by-day itinerary, checks each activity against the day's weather, runs a ReAct revision loop with a validation gate, and returns a Pydantic-validated `TravelPlan` object.

## Lesson index

### Prompt design

- [Giving the agent a clear role](./giving-the-agent-a-clear-role.md) — why persona assignment is the first move, not an afterthought.
- [Aligning prompt examples with your schema](./aligning-prompt-examples-with-schema.md) — what happens when your worked example uses `destination` and your model expects `city`.

### Tool definition

- [Documenting date formats in tool docstrings](./date-format-in-tool-docstrings.md) — the YYYY-MM-DD rabbit hole, and why "date (str): the date" never works.
- [Declaring all required tool parameters](./declaring-all-required-tool-parameters.md) — every parameter the function needs must appear in the schema, no exceptions.

### ReAct loop

- [Action JSON format in ReAct prompts](./react-action-json-format.md) — `{"tool_name": ..., "arguments": {...}}` vs free-form prose, and what the runner can actually parse.
- [Validation gates before the final answer](./validation-gates-before-final-answer.md) — why "run evals before finalizing" needs to be a hard gate, not a suggestion.
- [Terminating ReAct loops explicitly](./terminating-react-loops.md) — `final_answer_tool` as the only exit, and what happens when it isn't.

### Submission discipline

- [Submitting a runnable notebook](./submitting-a-runnable-notebook.md) — saved cell outputs are evidence; without them, even correct code fails review.

## Project shape (for context)

The project has five evaluation criteria mapped to specific code surfaces:

| # | Surface | What it tests |
|---|---------|---------------|
| 1 | `ITINERARY_AGENT_SYSTEM_PROMPT` | Role, context injection, schema embedding, CoT scaffolding |
| 2 | `ACTIVITY_AND_WEATHER_ARE_COMPATIBLE_SYSTEM_PROMPT` | Few-shot examples, output label format, edge-case coverage |
| 3 | `get_activities_by_date_tool` docstring | Parameter docs, format strings, purpose clarity |
| 4 | `ITINERARY_REVISION_AGENT_SYSTEM_PROMPT` | THOUGHT/ACTION/OBSERVATION cycle, tools list, exit gate, validation sequencing |
| 5 | `VacationInfo` / `TravelPlan` Pydantic models | Schema-in-prompt, model_validate flow, end-to-end display |
