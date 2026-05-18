# Udacity Reviews — Engineering Notes

Personal notes on AI agent engineering — prompt design, tool definitions, ReAct loops, structured output, Pydantic schema injection — written while mentoring agent-building projects.

Each page is a focused micro-essay on **one concept**. The structure is consistent: what the lesson is, why it matters in production, a minimal code example, common pitfalls, and links to first-party sources for deeper reading.

These notes are referenced inline in project feedback when a specific concept needs reinforcement.

## Contents

- **[travel-agent/](./travel-agent/)** — itinerary-planning agent with weather-aware activity selection, ReAct revision loop, Pydantic-validated structured output.

More projects to come.

## Why a separate repo

Vendor documentation moves, gets reshuffled, occasionally returns 403 to bot fetchers, and sometimes only adjacently covers the specific failure mode that came up in a review. These notes are the controlled-content layer: the lesson stays put, links can be refreshed in one place, and each page is written for the specific failure pattern it addresses rather than as general API reference.

## Conventions

- Code examples in Python, OpenAI / Anthropic SDK idioms where applicable.
- "Further reading" links favor first-party vendor docs (OpenAI, Anthropic, Pydantic, etc.) over third-party tutorials.
- No course platforms in the link list (Udemy, DataCamp, etc.).

— Sharad
