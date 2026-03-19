# Research Agent

An interactive AI research agent that autonomously searches the web, reads articles, and produces structured Markdown reports. Built with a custom ReAct loop using the OpenAI SDK directly (no LangChain/LangGraph).

## Setup

### Option 1: Docker (recommended)

```bash
cp .env.example .env
# Edit .env and add your API key

docker compose up --build
# Open http://localhost:8000
```

### Option 2: Local

```bash
# 1. Create a virtual environment
python -m venv .venv
source .venv/bin/activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Configure environment
cp .env.example .env
# Edit .env and add your API key

# 4. Run (pick one)
uvicorn app:app --reload  # Web UI at http://localhost:8000
python main.py            # Console REPL
```

### Environment Variables

| Variable     | Description                                      | Example          |
|-------------|--------------------------------------------------|------------------|
| `API_KEY`   | OpenAI API key                                   | `sk-...`         |
| `MODEL_NAME`| Model identifier                                 | `gpt-4.1-mini`  |
| `BASE_URL`  | OpenAI-compatible API endpoint (optional)        | `https://api.openai.com/v1` |

`BASE_URL` allows pointing the agent at any OpenAI-compatible provider (Azure OpenAI, local LLMs via Ollama/vLLM, etc.).

## Usage

### Web UI (FastAPI)

Open `http://localhost:8000` — type a question, watch the agent research in real time via SSE streaming. Token usage is displayed in the sidebar. Click "New Session" to reset conversation history and token counters.

### Console

```
Research Agent v0.1.0 (type 'exit' to quit)
Model: gpt-4.1-mini
----------------------------------------

You: Compare three RAG approaches: naive, sentence-window, and parent-child retrieval

  [web_search]("naive RAG approach explained") — 5 results found
  [web_search]("sentence window retrieval RAG") — 5 results found
  [read_url]("https://example.com/rag-comparison") — extracted 8,000 chars
  [write_report]("rag_comparison") — Report saved successfully to: output/2026-03-19_0628_rag_comparison.md

Agent: I've completed the research and saved a detailed comparison report.

You: Now focus on the parent-child approach — what are the best practices?

Agent: [remembers previous context, does more research]

You: exit
Goodbye!
```

## Architecture

```
research-agent/
├── app.py            # FastAPI web UI (SSE streaming, New Session, reports panel)
├── main.py           # Console REPL
├── agent.py          # Custom ReAct loop (no LangChain/LangGraph)
├── tools.py          # Tool functions + JSON Schema definitions
├── config.py         # Settings (pydantic-settings) and system prompt
├── requirements.txt  # Pinned dependencies (no LangChain)
├── VERSION           # Single source of truth for app version
├── Dockerfile        # Multi-stage container build
├── docker-compose.yml
├── .env.example      # Environment variable template
├── example_output/   # Sample generated report
│   └── report.md
├── output/           # Agent-generated reports (tracked in git)
├── logs/             # agent.log (rotating, tracked in git)
├── ARCHITECTURE.md   # Code flow and architecture explanation
└── DEVLOG.md         # Development changelog
```

### Agent Loop

The agent uses a custom ReAct (Reason + Act) while-loop, calling the OpenAI API directly:

1. **Reason** — The LLM analyzes the query and decides what to do
2. **Act** — If the response contains `tool_calls`, execute them via `TOOL_FUNCTIONS` dispatch dict
3. **Observe** — Tool results are appended to the `messages` list as `role: "tool"` entries
4. **Repeat** — Loop continues until the LLM responds without tool calls (final answer)
5. **Safety** — `max_iterations` (default: 25) prevents infinite loops

### Tools

| Tool           | Purpose                                         |
|---------------|--------------------------------------------------|
| `web_search`  | Search DuckDuckGo for relevant sources           |
| `read_url`    | Extract full text from a web page (truncated)    |
| `write_report`| Save the final Markdown report to `output/`      |
| `list_reports`| List previously saved reports (newest first)     |
| `read_file`   | Read a saved report from `output/`               |

Tools are defined as:
- **Plain Python functions** (no `@tool` decorator)
- **JSON Schema dicts** (`TOOL_SCHEMAS`) in OpenAI's tool-calling format
- **Dispatch dict** (`TOOL_FUNCTIONS`) mapping tool names to callables

### Context Engineering

- `read_url` truncates page text to 8,000 characters to prevent context window overflow
- `web_search` returns only titles, URLs, and snippets (not full pages)
- `read_file` truncates to the same limit
- `max_iterations` (default: 25) prevents the agent from looping indefinitely

### System Prompt Engineering

The system prompt uses structured techniques from the course lecture:
- **XML-tagged sections** (`<role>`, `<tools>`, `<strategy>`, `<rules>`, `<example>`, `<edge_cases>`)
- **Chain-of-thought** instructions (step-by-step reasoning)
- **Few-shot example** showing a complete tool-call sequence
- **Negative constraints** (what NOT to do)
- **Structured output template** (report format)
- **Edge case handling** (vague queries, no results, iteration limit)

### Memory

Conversation memory is a plain `messages: list[dict]` — the full chat history is passed to the LLM on every call. No external memory store (MemorySaver, database). Memory persists across turns within a session but is lost on restart.

### Key Difference from Lesson 3

| Lesson 3 (LangChain)                  | Lesson 4 (Custom)                            |
|----------------------------------------|----------------------------------------------|
| `create_react_agent` from LangGraph    | Custom `while` loop in `agent.py`            |
| `@tool` decorator                      | JSON Schema dicts (`TOOL_SCHEMAS`)           |
| `MemorySaver` checkpointer            | Plain `messages: list[dict]`                 |
| `langchain-openai` `ChatOpenAI`       | Direct `openai.OpenAI` SDK                   |
| 3 framework packages                  | 1 SDK package (`openai`)                     |
| 21 lines in `agent.py`                | 244 lines in `agent.py`                      |
