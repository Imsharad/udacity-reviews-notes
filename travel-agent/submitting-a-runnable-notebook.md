# Submitting a runnable notebook

Saved cell outputs are evidence. Without them, the most carefully-written code might as well not exist — a reviewer can't verify the agent works, the loop terminates, the schema validates, or the API calls return the expected shape. Even a perfect implementation fails review if the notebook arrives unexecuted.

## Why it matters

A Jupyter notebook is two things at once: source code and a record of past executions. The `.ipynb` JSON contains both the cells you wrote and the outputs they produced — `execution_count`, stdout, return values, error tracebacks, rich displays. When you submit a notebook with `execution_count: null` for every cell (i.e., never run), you've submitted source code without proof it works.

The reviewer's job is to verify the rubric — does the agent run the THOUGHT/ACTION/OBSERVATION loop, does it call `run_evals_tool`, does the final `TravelPlan` validate? Those are all behavioral checks. Behavior is observable in outputs, not in source. Without outputs, the most a reviewer can confirm is "the code parses." That's not enough to pass a behavioral rubric.

Two adjacent failure modes look similar from the outside:

1. **Notebook never executed end-to-end**: cells run individually during development, but the final saved state has stale outputs (or no outputs) from earlier code versions. The reviewer sees inconsistency and can't verify the current code.
2. **Wrong notebook submitted**: a project from a different course (a video-game research agent, a finance tool) gets zipped and submitted instead of the travel agent. The reviewer can't evaluate what isn't there.

Both reduce to the same outcome: nothing to evaluate.

## The pattern

Before submitting:

1. **Kernel → Restart and Run All** (or equivalent). This forces a clean execution from cell 1 to the last cell with a fresh Python session.
2. **Verify all cells have `execution_count` filled** — visible as `[1]`, `[2]`, ... in the notebook gutter. Any cell still showing `[ ]` was skipped.
3. **Confirm the final cells produce visible output** — a printed `TravelPlan` object, a successful `model_validate_json` call, a saved ReAct trace showing the loop terminating via `final_answer_tool`.
4. **Save the notebook after the run** — Jupyter sometimes loses outputs if you close before saving. The on-disk `.ipynb` is what gets submitted; the in-memory state is not.
5. **Open the saved file in a fresh editor window** to spot-check: do you see the outputs that were on screen, or are the output sections empty?
6. **Verify the right file is in the submission zip** — `unzip -l submission.zip` or open the zip to confirm `project_starter.ipynb` is present and is the version with outputs.

```bash
# Quick sanity check before zipping
jupyter nbconvert --to script project_starter.ipynb  # parses ok?
ls -la project_starter.ipynb                          # recently modified?
python -c "import json; nb=json.load(open('project_starter.ipynb'));
           print(sum(1 for c in nb['cells']
                     if c.get('cell_type')=='code'
                     and c.get('execution_count') is None),
                 'unexecuted cells')"
```

If that last command prints anything other than `0 unexecuted cells`, you have skipped cells — re-run.

## Common pitfalls

- **Restart-and-run-all skipped after the final code change**: outputs are stale from a previous version. The reviewer sees code that says one thing and outputs that show another.
- **Submitting from a working copy with `.ipynb_checkpoints/`** — the checkpoint version (`project_starter-checkpoint.ipynb`) gets zipped instead of the main file. Always check the zip contents.
- **Wrong project zipped**: the submission contains a `UdaPlay/` directory or `finance_agent.ipynb` instead of the travel-agent notebook. The fix is to verify the file name matches the assignment's expected name.
- **Notebook executed, but the agent loop wasn't actually called at the end**: cells define functions and prompts, but the demonstration cell that actually runs the agent is at the bottom and was never executed. The reviewer sees definitions without behavior.

## Further reading

- [Jupyter Notebook documentation — Running notebooks](https://jupyter-notebook.readthedocs.io/en/stable/notebook.html#running-cells) — official guidance on cell execution and notebook state.
- [Pydantic — model_validate_json](https://docs.pydantic.dev/latest/concepts/json/) — the call to put at the end of your notebook to demonstrate end-to-end schema validation against real LLM output.
