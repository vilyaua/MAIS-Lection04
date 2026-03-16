# CLAUDE.md — Research Agent v2 (Custom ReAct Loop)

## Project Context

This is **homework-lesson-4** of the MULTI-AGENT-SYSTEMS course.
The goal: refactor the working Research Agent from `MAIS-Lection03/research-agent/` by **replacing all LangChain/LangGraph agent abstractions** with a hand-rolled ReAct loop and improved system prompt.

Reference implementation: `../MAIS-Lection03/research-agent/` (v0.4.3)
Task spec: `homework-lesson-4/README.md`

## What Must Change (Lesson-3 → Lesson-4)

| Lesson-3 (current)                    | Lesson-4 (target)                           |
|----------------------------------------|---------------------------------------------|
| `create_react_agent` from LangGraph    | Custom `while` loop driving the ReAct cycle |
| LangChain manages tool dispatch        | We parse `tool_calls` from API response     |
| `MemorySaver` for conversation memory  | Manual `messages: list[dict]` accumulation  |
| `@tool` decorator (LangChain)          | Tools described as JSON Schema for the API  |
| `langchain-openai` ChatOpenAI wrapper  | Direct `openai` SDK (`client.chat.completions.create`) |
| Basic system prompt (69 lines)         | Enhanced prompt with prompt-engineering techniques |

## Architecture (Target)

```
research-agent/
├── config.py          # Settings (pydantic-settings) + SYSTEM_PROMPT
├── tools.py           # Tool functions + JSON Schema definitions (no @tool)
├── agent.py           # Custom ReAct loop (no create_react_agent)
├── main.py            # Interactive REPL
├── app.py             # FastAPI web UI (keep SSE streaming)
├── requirements.txt   # Slimmed: openai, ddgs, trafilatura, pydantic-settings, fastapi, uvicorn
├── Dockerfile
├── docker-compose.yml
├── VERSION
└── output/            # Generated reports
```

## Implementation Plan

### 1. Dependencies — slim down

Remove: `langchain`, `langchain-openai`, `langgraph`
Add: `openai` (direct SDK)
Keep: `ddgs`, `trafilatura`, `pydantic`, `pydantic-settings`, `fastapi`, `uvicorn`

### 2. `tools.py` — JSON Schema + plain functions

Each tool is:
- A **plain Python function** (no `@tool` decorator)
- A **JSON Schema dict** conforming to OpenAI's tool-calling format

```python
# Pattern for each tool:
def web_search(query: str) -> str:
    """Implementation unchanged from lesson-3."""
    ...

WEB_SEARCH_SCHEMA = {
    "type": "function",
    "function": {
        "name": "web_search",
        "description": "Search the web via DuckDuckGo. Returns titles, URLs, and snippets.",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search query"}
            },
            "required": ["query"],
        },
    },
}

# Registry for dispatch:
TOOL_FUNCTIONS = {
    "web_search": web_search,
    "read_url": read_url,
    "write_report": write_report,
    "list_reports": list_reports,
    "read_file": read_file,
}

TOOL_SCHEMAS = [
    WEB_SEARCH_SCHEMA,
    READ_URL_SCHEMA,
    WRITE_REPORT_SCHEMA,
    LIST_REPORTS_SCHEMA,
    READ_FILE_SCHEMA,
]
```

### 3. `agent.py` — Custom ReAct Loop

The core change. Replace `create_react_agent` with:

```python
import json
from openai import OpenAI
from config import Settings, SYSTEM_PROMPT
from tools import TOOL_FUNCTIONS, TOOL_SCHEMAS

settings = Settings()
client = OpenAI(api_key=settings.api_key.get_secret_value())

def run_agent(user_input: str, messages: list[dict]) -> str:
    """
    Single-turn ReAct loop:
    1. Append user message to history
    2. Call LLM with tools
    3. If tool_calls in response → execute tools, append results, repeat
    4. If no tool_calls → return final text response
    5. Safety: max_iterations limit
    """
    messages.append({"role": "user", "content": user_input})

    for i in range(settings.max_iterations):
        response = client.chat.completions.create(
            model=settings.model_name,
            messages=messages,
            tools=TOOL_SCHEMAS,
        )
        choice = response.choices[0]
        assistant_msg = choice.message

        # Append assistant message to history
        messages.append(assistant_msg.model_dump())

        # If no tool calls → final answer
        if not assistant_msg.tool_calls:
            return assistant_msg.content

        # Execute each tool call
        for tool_call in assistant_msg.tool_calls:
            name = tool_call.function.name
            args = json.loads(tool_call.function.arguments)

            # Log: 🔧 Tool call: name(args)
            print(f"🔧 Tool call: {name}({args})")

            try:
                result = TOOL_FUNCTIONS[name](**args)
            except Exception as e:
                result = f"Error: {e}"

            # Log: 📎 Result: ...
            print(f"📎 Result: {result[:200]}")

            # Append tool result to history
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result,
            })

    return "I hit the maximum number of reasoning steps. Please try a simpler query."
```

**Key points:**
- `messages` list is passed in and mutated — this IS the memory (no MemorySaver)
- The loop continues until the LLM responds without `tool_calls` or hits `max_iterations`
- Tool errors are caught and returned as strings (LLM adapts)

### 4. `main.py` — REPL with manual memory

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

while True:
    user_input = input("\nYou: ").strip()
    if user_input.lower() in {"exit", "quit", "q"}:
        break
    response = run_agent(user_input, messages)
    print(f"\nAgent: {response}")
```

Memory = the `messages` list persists across turns within the session.

### 5. `app.py` — Adapt for streaming

For the web UI, create a streaming variant of the ReAct loop that yields chunks.
The sync→async bridge pattern (run_in_executor + Queue) from lesson-3 still applies, but the inner loop is now our custom code instead of `agent.stream()`.

### 6. System Prompt — Enhanced with prompt engineering

Apply these techniques from the lecture:
- **Clear role definition** with persona and boundaries
- **Structured sections** (XML tags or Markdown headers for clarity)
- **Step-by-step reasoning instructions** (chain-of-thought)
- **Few-shot examples** showing expected tool-call sequences
- **Negative constraints** (what NOT to do)
- **Output format spec** with concrete template
- **Fallback behavior** for edge cases

### 7. Logging

Console output format (match the spec):
```
You: Порівняй naive RAG та sentence-window retrieval

🔧 Tool call: web_search(query="naive RAG approach explained")
📎 Result: Found 5 results...

🔧 Tool call: web_search(query="sentence window retrieval RAG")
📎 Result: Found 5 results...

🔧 Tool call: read_url(url="https://example.com/rag-comparison")
📎 Result: [5000 chars] Article about RAG approaches...

🔧 Tool call: write_report(filename="rag_comparison.md", content="# RAG Comparison...")
📎 Result: Report saved to output/rag_comparison.md

Agent: Звіт збережено у output/rag_comparison.md. Ось основні відмінності: ...
```

Plus file logging with `RotatingFileHandler` (keep from lesson-3).

### 8. Error Handling

- Tool exceptions → caught, returned as error string to LLM
- API errors (auth, rate limit, connection) → caught in REPL, logged, session continues
- Iteration limit → graceful message instead of infinite loop
- Invalid tool name → return error string, don't crash

## Files to Copy As-Is from Lesson-3

- `Dockerfile` (adjust ENTRYPOINT if needed)
- `docker-compose.yml`
- `.env.example`
- `.gitignore`
- `.dockerignore`
- `VERSION` (reset to `0.1.0`)
- `example_output/report.md`

## Files to Rewrite

- `config.py` — keep Settings, rewrite SYSTEM_PROMPT
- `tools.py` — keep function bodies, replace `@tool` with JSON Schema
- `agent.py` — full rewrite (custom ReAct loop)
- `main.py` — simplify (no LangChain imports, direct loop)
- `app.py` — adapt streaming to custom loop
- `requirements.txt` — remove langchain/langgraph, add openai

## Conventions

- Python 3.12+
- Ruff for linting/formatting (line-length=100, same rules as lesson-3)
- Pre-commit hooks (copy `.pre-commit-config.yaml` from lesson-3 root)
- DEVLOG.md — update after every significant change
- VERSION file — single source of truth, start at `0.1.0`

## Acceptance Criteria

1. `python main.py` starts interactive REPL
2. No imports from `langchain`, `langgraph`, or any agent framework
3. Tools defined as JSON Schema, dispatched manually
4. Conversation memory via `messages` list (multi-turn works)
5. Console shows each tool call with args and results
6. Agent makes 3-5+ tool calls per research query
7. Reports saved to `output/` via `write_report`
8. Errors don't crash the agent
9. Max iteration limit prevents infinite loops
10. System prompt uses prompt engineering techniques from lecture
11. FastAPI web UI works with SSE streaming
12. Docker build works
