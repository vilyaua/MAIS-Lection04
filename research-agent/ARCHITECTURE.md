# Research Agent — Architecture & Code Flow

## High-level Architecture

```
User (CLI or Browser)
    |
    v
+---------------+     +-------------+     +----------------+
|  main.py      |     |  agent.py   |     |   tools.py     |
|  (CLI REPL)   |---->|  (Custom    |---->|  web_search    |
|               |     |   ReAct     |     |  read_url      |
|  app.py       |---->|   loop)     |<----|  write_report  |
|  (FastAPI)    |     +-------------+     |  list_reports  |
+---------------+           |             |  read_file     |
                            v             +----------------+
                      +-------------+
                      |   OpenAI    |
                      |   API       |
                      +-------------+
```

## Key Difference from Lesson 3

Lesson 3 used LangChain/LangGraph (`create_react_agent`, `MemorySaver`, `@tool`).
This version replaces all of that with:

- **Direct OpenAI SDK** (`openai.OpenAI`) instead of `ChatOpenAI`
- **Custom while-loop** instead of `create_react_agent`
- **`messages: list[dict]`** instead of `MemorySaver`
- **JSON Schema dicts** instead of `@tool` decorators

## Startup Flow

1. **`config.py`** loads first — reads `.env` via pydantic-settings, exposes `Settings` and `SYSTEM_PROMPT`.

2. **`tools.py`** defines 5 plain functions + `TOOL_SCHEMAS` (OpenAI tool-calling format) + `TOOL_FUNCTIONS` dispatch dict.

3. **`agent.py`** provides `create_client()` and two ReAct loop functions:
   - `run_agent_turn()` — returns final answer string
   - `run_agent_turn_streaming()` — yields events (for SSE)

## The ReAct Loop (core logic)

```
User message
    |
    v
+----------------------------------------------+
|  1. Append user message to messages list     |
|                                               |
|  2. Call client.chat.completions.create()     |
|     with messages + tool schemas              |
|                                               |
|  3. If response has tool_calls:              |
|     a. Serialize assistant msg to history     |
|     b. Execute each tool via TOOL_FUNCTIONS   |
|     c. Append tool results to history         |
|     d. Go to step 2                           |
|                                               |
|  4. If no tool_calls → return final answer   |
|                                               |
|  5. Safety: max_iterations limit (25)         |
+----------------------------------------------+
    |
    v
Final text response to user
```

## Memory

The `messages` list IS the memory. It persists across turns within a session:

- CLI: single `messages` list lives for the entire REPL session
- Web: `web_messages` module-level list persists across HTTP requests (in-process only)

No external memory store — lost on restart.

## Two Interfaces, Same Loop

**`main.py` (CLI)** — REPL using `run_agent_turn_streaming()`:
```
input() → run_agent_turn_streaming() → print events → loop
```

**`app.py` (FastAPI)** — SSE using the same streaming variant.
Bridge pattern: `run_in_executor` runs the sync generator in a thread,
events flow through an `asyncio.Queue` to the async SSE generator.

## System Prompt Engineering

The prompt uses several techniques from the lecture:
- **XML-tagged sections** (`<role>`, `<tools>`, `<strategy>`, `<rules>`, `<example>`, `<edge_cases>`)
- **Chain-of-thought** instructions (step-by-step reasoning)
- **Few-shot example** showing a complete tool-call sequence
- **Negative constraints** (what NOT to do)
- **Structured output template** (report format)
- **Edge case handling** (vague queries, no results, iteration limit)

## File Summary

| File | Role |
|------|------|
| `config.py` | Settings from `.env` + system prompt |
| `tools.py` | 5 tool functions + JSON Schema definitions |
| `agent.py` | Custom ReAct loop (no LangChain) |
| `main.py` | CLI REPL interface |
| `app.py` | FastAPI web UI with SSE |
| `Dockerfile` | Multi-stage container build |
