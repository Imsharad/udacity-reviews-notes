# Submitting a runnable notebook

The reviewer opens the notebook, presses Restart and Run All, and expects to see every cell produce real output. If any cell errors, if any output is missing, or if the notebook is not the one the rubric expects, the criterion fails before the substantive evaluation begins.

This is the most common reason a UdaPlay submission fails on RAG (criterion 1) and on agent demonstration (criterion 4). The model didn't do anything wrong; the notebook didn't get re-executed before submission.

## Why it matters

A notebook is not a Python file. The outputs the reviewer sees are the outputs that were on disk when the file was last saved. If you cleared outputs to keep the diff clean, or if you edited a cell and didn't re-run it, the output state diverges from the code state. The reviewer sees stale output, no output, or contradictory output.

For agent projects specifically, this matters more than for pure-code projects. The criteria explicitly check for evidence of agent execution: tool calls, reasoning traces, citations, multiple query demonstrations. All of that lives in the notebook's saved outputs. Stripping them strips the evidence.

## The pattern

Before submission, every time:

```text
1. Kernel -> Restart Kernel and Run All Cells
2. Wait for every cell to finish (no [*] indicators left)
3. Scroll through the entire notebook and verify each cell has output
4. Confirm the cells the rubric mentions are populated:
   - For Part 01 (RAG): semantic search query results visible
   - For Part 02 (Agent): three example queries with full traces
5. Save the notebook (Ctrl/Cmd + S) AFTER step 4
6. Confirm the .ipynb file mtime updated
7. Zip the project and submit
```

The "save AFTER step 4" step is the one most often skipped — the notebook is saved before the full run finishes, freezing partial output state.

## Common pitfalls

- **Outputs cleared "to keep the diff clean"**: the project gets submitted with no execution evidence. Reviewer flags as incomplete.
- **Cells run individually, out of order**: the notebook works locally because the kernel has state from earlier cells, but on a fresh kernel the imports or variables aren't there. Restart-and-Run-All catches this.
- **The wrong notebook submitted**: a student working with both `*_starter_*.ipynb` and `*_solution_*.ipynb` zips the starter by mistake. The grader picks the file with the more recent mtime or the most-populated outputs, but if you submit only the starter, it has no work to grade.
- **API keys not exported in the environment that runs the notebook**: cells silently catch the exception and return empty results. The notebook looks complete but every LLM/web-search call returned None.
- **Notebook saved with a `[*]` cell**: a long-running cell that was interrupted leaves an in-flight indicator. The output is partial; the next cell may have run with stale state.
- **Submission zip contains a stale `.ipynb_checkpoints/` copy**: most graders ignore this, but some pick the checkpoint over the live file. Delete checkpoint dirs before zipping.

## Further reading

- [Jupyter — Saving Notebooks](https://jupyterlab.readthedocs.io/en/stable/user/files.html) — official guidance on save behavior and checkpoint files.
- [nbformat — Notebook structure](https://nbformat.readthedocs.io/en/latest/format_description.html) — what the .ipynb file actually stores and why cleared outputs disappear from the saved file.
