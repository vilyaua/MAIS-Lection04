# Lesson 3 vs Lesson 4 — Detailed Comparison

> Every change between `MAIS-Lection03/research-agent/` (v0.4.3, LangChain/LangGraph) and `MAIS-Lection04/research-agent/` (v0.1.0, Custom ReAct Loop).

---

## 1. `requirements.txt` — Dependency Overhaul

**What changed:** Removed all LangChain/LangGraph packages, added direct OpenAI SDK.

```diff
- langchain>=1.2.0
- langchain-openai>=0.3.0
- langgraph>=0.4.0
+ openai>=1.70.0
  ddgs>=7.0
  trafilatura>=2.0.0
  pydantic>=2.12.0
  pydantic-settings>=2.12.0
  fastapi>=0.115.0
  uvicorn>=0.34.0
```

**Why:** The core assignment — replace the framework abstraction with hand-rolled code. `openai` SDK talks to the API directly; `langchain`/`langchain-openai`/`langgraph` are no longer needed since the ReAct loop, tool dispatch, and memory are all custom now.

---

## 2. `VERSION`

```diff
- 0.4.3
+ 0.1.0
```

**Why:** Fresh project, new major version lineage starting from scratch.

---

## 3. `config.py` — Settings + System Prompt

### 3a. Settings class — added `base_url`

```diff
  class Settings(BaseSettings):
      api_key: SecretStr
      model_name: str = "gpt-4.1-mini"
+     base_url: str | None = None  # set to use an OpenAI-compatible provider

      max_search_results: int = 5
-     max_url_content_length: int = 8000  # context engineering: caps text fed back to LLM
+     max_url_content_length: int = 8000  # caps text fed back to LLM
      output_dir: str = "output"
-     max_iterations: int = 25  # LangGraph recursion_limit — prevents infinite ReAct loops
+     max_iterations: int = 25  # prevents infinite ReAct loops
```

**Why:** `base_url` allows pointing at OpenAI-compatible providers (local LLMs, Azure, etc.) — the direct SDK makes this trivial. Comment changed because it's no longer LangGraph's recursion limit.

### 3b. System Prompt — Complete Rewrite with Prompt Engineering

**Lesson 3** (69 lines, plain Markdown):
```
You are a Research Agent — an AI assistant that investigates topics...

## Your Tools
1. **web_search(query)** — ...
...
## Important Rules
- Always make at least 3 tool calls...
```

**Lesson 4** (120 lines, XML-tagged sections with prompt engineering techniques):
```xml
<role>
You are a Research Agent — an autonomous AI assistant...
You operate in a ReAct loop: on every turn you either call a tool or produce a
final answer.  Think step-by-step before choosing an action.
</role>

<tools>...</tools>
<strategy>...</strategy>
<output_format>...</output_format>
<rules>...</rules>
<example>...</example>
<edge_cases>...</edge_cases>
```

**Key differences in the system prompt:**

| Technique | Lesson 3 | Lesson 4 |
|-----------|----------|----------|
| Structure | Flat Markdown headers | XML tags (`<role>`, `<tools>`, `<strategy>`, `<rules>`, `<example>`, `<edge_cases>`) |
| Role definition | One sentence | Explicit persona + ReAct loop awareness |
| Chain-of-thought | Not present | "Think step-by-step before choosing an action" |
| Few-shot example | Not present | Full example with tool call sequence for "Compare React and Vue" |
| Negative constraints | 1 rule ("If a tool call fails, adapt") | 4 rules: no hallucinated URLs, no read_url without web_search, no repeated failed calls, adapt on failure |
| Output format | Bullet list | Fenced code block template showing exact structure |
| Edge cases | Not present | Dedicated `<edge_cases>` section: vague queries, no results, iteration limit |
| Fallback behavior | Not present | "Save partial findings if hitting iteration limit" |

**Why:** Applies prompt engineering techniques from the lecture — structured sections, chain-of-thought, few-shot examples, negative constraints, output format spec, and edge case handling.

---

## 4. `tools.py` — From `@tool` Decorator to JSON Schema

### 4a. Imports — removed LangChain

```diff
- import trafilatura
- from ddgs import DDGS
- from langchain_core.tools import tool
+ import trafilatura
+ from ddgs import DDGS
```

**Why:** `@tool` decorator from `langchain_core.tools` is replaced by explicit JSON Schema dicts.

### 4b. Tool function declarations — removed `@tool` decorator

Each function lost its `@tool` decorator and had its docstring simplified (the detailed descriptions moved into JSON Schema):

```diff
- @tool
- def web_search(query: str) -> str:
-     """Search the web using DuckDuckGo. Returns a list of results with title, URL, and snippet for each.
-
-     Args:
-         query: The search query string.
-     """
+ def web_search(query: str) -> str:
+     """Search the web using DuckDuckGo."""
```

Same pattern applied to all 5 tools: `web_search`, `read_url`, `write_report`, `list_reports`, `read_file`.

**Why:** With LangChain, the `@tool` decorator auto-generates the tool schema from the function signature + docstring. Without LangChain, tool descriptions live in explicit JSON Schema dicts — so the docstrings can be shorter (they're just for developers now, not for the LLM).

### 4c. Function bodies — unchanged

All 5 tool function implementations are **identical** between Lesson 3 and Lesson 4. The business logic (DuckDuckGo search, trafilatura extraction, file I/O) didn't change.

### 4d. Added JSON Schema definitions + dispatch registry

Lesson 4 adds ~120 new lines that didn't exist in Lesson 3:

```python
# OpenAI tool-calling schemas
TOOL_SCHEMAS = [
    {
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "Search the web using DuckDuckGo...",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "The search query string."}
                },
                "required": ["query"],
                "additionalProperties": False,
            },
        },
    },
    # ... same pattern for read_url, write_report, list_reports, read_file
]

# Dispatch dict: tool name -> callable
TOOL_FUNCTIONS = {
    "web_search": web_search,
    "read_url": read_url,
    "write_report": write_report,
    "list_reports": list_reports,
    "read_file": read_file,
}
```

**Why:** The OpenAI API needs tool definitions as JSON Schema (passed to `tools=` parameter). `TOOL_FUNCTIONS` dict enables dispatching tool calls by name string — replacing LangChain's automatic tool routing. In Lesson 3, `@tool` + `create_react_agent(tools=[...])` handled both schema generation and dispatch automatically.

---

## 5. `agent.py` — Full Rewrite (Core Change)

This is the most significant change. The entire file was rewritten.

### Lesson 3 (21 lines — framework wiring):

```python
from langchain_openai import ChatOpenAI
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import create_react_agent
from config import SYSTEM_PROMPT, Settings
from tools import list_reports, read_file, read_url, web_search, write_report

settings = Settings()
llm = ChatOpenAI(model=settings.model_name, api_key=settings.api_key.get_secret_value())
tools = [web_search, read_url, write_report, list_reports, read_file]
memory = MemorySaver()
agent = create_react_agent(model=llm, tools=tools, checkpointer=memory, prompt=SYSTEM_PROMPT)
```

### Lesson 4 (244 lines — custom ReAct loop):

```python
from openai import OpenAI
from config import Settings
from tools import TOOL_FUNCTIONS, TOOL_SCHEMAS

def create_client(settings: Settings) -> OpenAI:
    return OpenAI(api_key=settings.api_key.get_secret_value(), base_url=settings.base_url)

def _execute_tool_call(tool_call) -> tuple[str, str, dict, str]:
    """Execute a single tool call — never raises."""
    ...

def run_agent_turn(user_input, messages, settings, client) -> tuple[str, dict]:
    """Single-turn ReAct loop returning final answer + usage."""
    messages.append({"role": "user", "content": user_input})
    for _iteration in range(settings.max_iterations):
        response = client.chat.completions.create(
            model=settings.model_name, messages=messages, tools=TOOL_SCHEMAS,
        )
        # ... serialize assistant message, execute tool calls, append results
        if not assistant_msg.tool_calls:
            return assistant_msg.content, usage_totals
    return "max iterations reached", usage_totals

def run_agent_turn_streaming(user_input, messages, settings, client):
    """Streaming variant — yields events for web UI SSE."""
    # Same loop but yields {"type": "tool_result"/"message"/"tokens"/"done"} dicts
```

**What replaced what:**

| LangChain/LangGraph (L3) | Custom code (L4) | Explanation |
|---------------------------|-------------------|-------------|
| `ChatOpenAI(model=..., api_key=...)` | `OpenAI(api_key=..., base_url=...)` | Direct SDK client instead of LangChain wrapper |
| `create_react_agent(model, tools, checkpointer, prompt)` | `while` loop in `run_agent_turn()` | The ReAct loop is now explicit: call LLM → check for tool_calls → execute → append → repeat |
| `MemorySaver()` | `messages: list[dict]` passed as argument | Memory is just a plain list of message dicts, mutated in place |
| `agent.stream({"messages": [...]}, config=config)` | `run_agent_turn_streaming()` generator | Yields typed event dicts instead of LangGraph chunks |
| LangGraph's automatic tool dispatch | `_execute_tool_call()` + `TOOL_FUNCTIONS[name](**args)` | Manual JSON parsing of `tool_call.function.arguments` + dict lookup |
| `GraphRecursionError` | `for _iteration in range(max_iterations)` + graceful message | Loop limit instead of framework exception |
| `msg.model_dump()` for serialization | Manual dict construction with `tool_calls` list | Explicit serialization of assistant messages to avoid SDK object issues |
| Singleton `agent` module-level object | `create_client()` factory function | Client created explicitly, not at import time |

### New: `_execute_tool_call()` helper

Handles three failure modes that LangChain handled internally:
1. **JSON parse error** — malformed `function.arguments` from the LLM
2. **Unknown tool name** — returns error string instead of KeyError crash
3. **Tool execution error** — catches any exception, returns error string

### New: Two execution variants

- `run_agent_turn()` — returns final answer string (simpler, used for testing)
- `run_agent_turn_streaming()` — yields event dicts with types: `tool_call`, `tool_result`, `message`, `tokens`, `done`

### New: Explicit message serialization

```python
msg_dict = {"role": "assistant"}
if assistant_msg.content:
    msg_dict["content"] = assistant_msg.content
if assistant_msg.tool_calls:
    msg_dict["tool_calls"] = [
        {"id": tc.id, "type": "function", "function": {"name": tc.function.name, "arguments": tc.function.arguments}}
        for tc in assistant_msg.tool_calls
    ]
messages.append(msg_dict)
```

**Why:** The OpenAI SDK returns Pydantic objects. Appending them directly to the messages list can cause serialization issues on the next API call. Manual dict construction ensures clean JSON-serializable messages in the conversation history.

---

## 6. `main.py` — Simplified REPL

### Imports

```diff
- from langgraph.errors import GraphRecursionError
  from openai import APIConnectionError, APIError, AuthenticationError, RateLimitError

- from agent import agent
- from config import APP_VERSION, Settings
+ from agent import create_client, run_agent_turn_streaming
+ from config import APP_VERSION, SYSTEM_PROMPT, Settings
```

**Why:** No more LangGraph imports. Now imports `create_client` and `run_agent_turn_streaming` from the custom agent module. Also imports `SYSTEM_PROMPT` directly (was embedded in agent setup before).

### Memory initialization

```diff
- # LangGraph config with thread_id for MemorySaver
- config = {
-     "configurable": {"thread_id": "session-1"},
-     "recursion_limit": settings.max_iterations,
- }
+ client = create_client(settings)
+ messages: list[dict] = [{"role": "system", "content": SYSTEM_PROMPT}]
```

**Why:** Memory is now just a `messages` list initialized with the system prompt. No more `MemorySaver` thread configuration. The `client` is created once and reused.

### Main loop — streaming events instead of LangGraph chunks

```diff
- # LangGraph chunk processing
- for chunk in agent.stream({"messages": [("user", user_input)]}, config=config):
-     if "agent" in chunk and "messages" in chunk["agent"]:
-         for msg in chunk["agent"]["messages"]:
-             if hasattr(msg, "content") and msg.content:
-                 print(f"\nAgent: {msg.content}")
-             if hasattr(msg, "tool_calls") and msg.tool_calls:
-                 for tc in msg.tool_calls:
-                     pending_calls[tc["id"]] = _get_tool_call_args(tc["name"], tc["args"])
-             if hasattr(msg, "usage_metadata") and msg.usage_metadata:
-                 ...
-     if "tools" in chunk and "messages" in chunk["tools"]:
-         for msg in chunk["tools"]["messages"]:
-             call_args = pending_calls.pop(...)
-             print(_format_tool_status(msg, call_args))

+ # Custom event processing
+ for event in run_agent_turn_streaming(user_input, messages, settings, client):
+     if event["type"] == "tool_result":
+         print(_format_tool_status(event["name"], event["args"], event["result"]))
+     elif event["type"] == "message":
+         print(f"\nAgent: {event['content']}")
+     elif event["type"] == "tokens":
+         session_tokens.update(event["data"])
```

**Why:** LangGraph chunks use a nested dict structure (`chunk["agent"]["messages"]`, `chunk["tools"]["messages"]`) with LangChain message objects. The custom loop yields flat typed event dicts — simpler to consume, no need to pair tool calls with results across separate chunks.

### `_format_tool_status` — signature change

```diff
- def _format_tool_status(msg, call_args: str = "") -> str:
-     name = msg.name
-     content = msg.content
-     args_part = f'("{call_args}") ' if call_args else " "
+ def _format_tool_status(name: str, args: dict, result: str) -> str:
+     primary_arg = _get_tool_call_args(name, args)
+     args_part = f'("{primary_arg}") ' if primary_arg else " "
```

**Why:** In Lesson 3, this function received a LangChain `ToolMessage` object (with `.name` and `.content` attributes) + separately-tracked `call_args` string. In Lesson 4, it receives plain strings/dicts directly from the event — no LangChain objects involved.

### Error handling

```diff
- except GraphRecursionError:
-     print("\nAgent: Sorry, I hit the maximum number of steps.")
-     logger.warning("GraphRecursionError — recursion_limit=%d", settings.max_iterations)
```

**Why:** `GraphRecursionError` no longer exists. The custom loop handles iteration exhaustion internally by returning a graceful message — no exception to catch.

### Removed: `pending_calls` buffer

Lesson 3 needed a `pending_calls: dict[str, str]` to pair tool call args (from "agent" chunks) with tool results (from "tools" chunks) — they arrived in separate chunks. In Lesson 4, the streaming generator yields complete `tool_result` events with both args and results together, so no buffering is needed.

---

## 7. `app.py` — Adapted for Custom Streaming

### Imports

```diff
- from langgraph.errors import GraphRecursionError
- from agent import agent
- from config import APP_VERSION, Settings
+ from agent import create_client, run_agent_turn_streaming
+ from config import APP_VERSION, SYSTEM_PROMPT, Settings
```

### Session state — explicit message list instead of MemorySaver

```diff
- session_tokens = {"input": 0, "output": 0, "total": 0}
+ web_messages: list[dict] = [{"role": "system", "content": SYSTEM_PROMPT}]
+ web_client = create_client(settings)
+ session_tokens = {"input": 0, "output": 0, "total": 0}
```

**Why:** Web session memory is now the `web_messages` list (persists across requests within the process). The OpenAI client is created once at module level.

### `_format_tool_event` — signature change

```diff
- def _format_tool_event(msg, call_args: str = "") -> dict:
-     name = msg.name
-     content = msg.content
+ def _format_tool_event(name: str, args: dict, result: str) -> dict:
+     primary_arg = _get_tool_call_args(name, args)
```

**Why:** Same as `main.py` — receives plain values instead of LangChain `ToolMessage` objects.

### `_sync_stream` — simplified

```diff
- def _sync_stream(prompt: str, config: dict):
-     yield from agent.stream(
-         {"messages": [("user", prompt)]},
-         config=config,
-     )
+ def _sync_stream(prompt: str):
+     yield from run_agent_turn_streaming(prompt, web_messages, settings, web_client)
```

**Why:** No more `config` parameter (thread_id, recursion_limit). The custom streaming function takes the messages list and settings directly.

### `_stream_response` — simplified event processing

```diff
- async def _stream_response(prompt: str) -> AsyncGenerator[str, None]:
-     """Bridge sync LangGraph streaming into async SSE."""
-     config = {
-         "configurable": {"thread_id": "web-session"},
-         "recursion_limit": settings.max_iterations,
-     }
-     ...
-     pending_calls: dict[str, str] = {}
-     while True:
-         chunk = await queue.get()
-         ...
-         if "agent" in chunk and "messages" in chunk["agent"]:
-             for msg in chunk["agent"]["messages"]:
-                 # process content, tool_calls, usage_metadata...
-                 if hasattr(msg, "tool_calls") and msg.tool_calls:
-                     for tc in msg.tool_calls:
-                         pending_calls[tc["id"]] = _get_tool_call_args(tc["name"], tc["args"])
-                 if hasattr(msg, "usage_metadata") and msg.usage_metadata:
-                     ...
-         if "tools" in chunk and "messages" in chunk["tools"]:
-             for msg in chunk["tools"]["messages"]:
-                 call_args = pending_calls.pop(...)
-                 event = _format_tool_event(msg, call_args)
-                 ...

+ async def _stream_response(prompt: str) -> AsyncGenerator[str, None]:
+     """Bridge sync ReAct streaming into async SSE."""
+     ...
+     while True:
+         event = await queue.get()
+         ...
+         if event["type"] == "message":
+             yield f"data: {json.dumps(...)}\n\n"
+         elif event["type"] == "tool_result":
+             tool_event = _format_tool_event(event["name"], event["args"], event["result"])
+             yield f"data: {json.dumps(...)}\n\n"
+         elif event["type"] == "tokens":
+             ...
```

**Key differences:**
1. No `config` dict with `thread_id` / `recursion_limit`
2. No `pending_calls` buffer for pairing tool calls with results
3. Events are flat typed dicts instead of nested LangGraph chunks
4. `GraphRecursionError` catch removed — iteration limit handled inside the generator
5. Error handling uses the event's `type` field instead of tuple detection

### Error handling in producer thread

```diff
- except GraphRecursionError:
-     err = "Sorry, I hit the maximum number of steps."
-     loop.call_soon_threadsafe(queue.put_nowait, ("error", err))
- except (APIError, ...) as e:
-     loop.call_soon_threadsafe(queue.put_nowait, ("error", err))
+ except (APIError, ...) as e:
+     loop.call_soon_threadsafe(queue.put_nowait, {"type": "error", "content": err})
```

**Why:** Error sentinel changed from `("error", msg)` tuple to `{"type": "error", "content": msg}` dict — consistent with the event dict pattern. `GraphRecursionError` removed.

### Docstrings updated

```diff
- """Bridge sync LangGraph streaming into async SSE.
-
-  LangGraph's agent.stream() is synchronous...
+ """Bridge sync ReAct streaming into async SSE.
+
+  run_agent_turn_streaming() is synchronous...
```

---

## 8. Files Unchanged

| File | Notes |
|------|-------|
| `Dockerfile` | Identical — multi-stage build, same ENTRYPOINT |
| `docker-compose.yml` | Identical — same ports, volumes, env_file |
| `CHAT_HTML` (in app.py) | Identical — same CSS, JS, UI layout |
| `.gitignore` / `.dockerignore` | Identical |
| `example_output/report.md` | Identical sample report |
| Logging setup | Identical pattern (dual console + rotating file) |
| API endpoints | Identical routes (`/`, `/api/info`, `/api/chat`, `/api/reports`, `/api/reports/{filename}`) |

---

## 9. New Files in Lesson 4

| File | Purpose |
|------|---------|
| `.env.example` | Template with `API_KEY`, `MODEL_NAME`, `BASE_URL` (L3 had this too but `BASE_URL` is new) |
| `ARCHITECTURE.md` | Updated technical docs reflecting custom ReAct loop |
| `DEVLOG.md` | Changelog for the refactor |

---

## 10. Summary: What Each Abstraction Was Replaced With

```
┌─────────────────────────────────┬────────────────────────────────────────────┐
│ LangChain/LangGraph (L3)        │ Custom Code (L4)                           │
├─────────────────────────────────┼────────────────────────────────────────────┤
│ langchain-openai (ChatOpenAI)   │ openai SDK (OpenAI client)                 │
│ @tool decorator                 │ Plain functions + JSON Schema dicts         │
│ create_react_agent()            │ while loop in run_agent_turn()             │
│ MemorySaver (checkpointer)      │ messages: list[dict] (plain list)          │
│ agent.stream() chunks           │ Generator yielding typed event dicts       │
│ GraphRecursionError             │ for-loop with max_iterations               │
│ LangChain ToolMessage objects   │ Plain str/dict values                      │
│ Automatic tool schema gen       │ Explicit TOOL_SCHEMAS list                 │
│ Automatic tool dispatch         │ TOOL_FUNCTIONS dict + manual dispatch      │
│ config["thread_id"]             │ System prompt prepended to messages list   │
│ msg.usage_metadata              │ response.usage (OpenAI SDK native)         │
└─────────────────────────────────┴────────────────────────────────────────────┘
```

**Lines of code:**
- Lesson 3 `agent.py`: ~21 lines (framework does the work)
- Lesson 4 `agent.py`: ~244 lines (everything is explicit)
- Lesson 3 `tools.py`: ~100 lines (functions only, schema auto-generated)
- Lesson 4 `tools.py`: ~234 lines (functions + explicit JSON schemas + dispatch dict)

**Trade-off:** More code, but zero framework dependencies for the core agent loop. Full control over message history, tool dispatch, error handling, and streaming behavior.
