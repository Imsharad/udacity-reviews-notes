# Giving the agent a clear role

The first line of a system prompt should tell the model *what it is*. Not what to do — that comes next. A generic LLM with no role assignment is a different (worse) instrument than the same model told it's a travel planner, code reviewer, or weather-compatibility evaluator.

## Why it matters

Role assignment narrows the model's distribution. "You are an expert travel planner" doesn't just sound nicer than no preamble — it shifts the prior over output style, vocabulary, structure, and what the model treats as relevant context. The same prompt body, with and without a role line, will produce noticeably different outputs.

In agent systems, the cost of a missing role compounds across the loop. A revision agent without a defined role might suggest meta-changes ("perhaps you want a different itinerary structure?") instead of acting within its lane. A compatibility evaluator without a role might hedge and refuse to return a binary decision the parser is waiting for.

## The pattern

```python
ITINERARY_AGENT_SYSTEM_PROMPT = """
You are an expert travel planner. You build day-by-day itineraries
that respect the traveler's interests, the weather forecast, and the
destination's available activities.

Your output is always a structured JSON object matching the TravelPlan
schema. You do not produce free-form prose, follow-up questions, or
alternative suggestions.

For each day of the trip:
1. Consult the weather forecast for that date.
2. Select activities from the provided list that match the traveler's
   interests and the day's weather.
3. ...
"""
```

The structure: **role**, then **output contract**, then **task instructions**. Skipping the role line and starting straight at the task is the most common version of this mistake.

Roles can be narrow. For a weather-compatibility evaluator:

```python
ACTIVITY_AND_WEATHER_ARE_COMPATIBLE_SYSTEM_PROMPT = """
You are a weather-compatibility evaluator. Given a planned activity
and the forecast for the day, decide whether the activity is safe
and enjoyable in that weather.

Your output is always exactly one of two strings:
- IS_COMPATIBLE
- IS_INCOMPATIBLE
"""
```

The role *is* the constraint here. Calling it "a weather-compatibility evaluator" (not "an assistant that helps with weather") makes the binary output feel native to the role.

## Common pitfalls

- **Starting with the task, no role**: the prompt jumps to "Generate an itinerary that..." with no persona. Works on Claude 4.7 about 60% of the time, fails subtly in long contexts.
- **Role as a one-word tag**: "You are a travel agent." vs "You are an expert travel planner who builds detailed day-by-day itineraries with attention to weather and traveler interests." The longer form costs three seconds and pays back in tighter output.
- **Mixed-message roles**: "You are a helpful assistant that is also a travel planner." The "helpful assistant" carrier wave wins; the planner specialization gets diluted.
- **Role copied from a different project**: a notebook that started as a video-game research agent and was repurposed for travel planning will sometimes still open with "You are UdaPlay, an expert in video game industry research." This is more common than it should be.

## Further reading

- [Anthropic — Give Claude a role](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering#give-claude-a-role) — first-party guidance with side-by-side examples of role-on vs role-off output.
- [OpenAI Cookbook — Prompt structure](https://developers.openai.com/cookbook/) — recipes that consistently lead with a role definition.
