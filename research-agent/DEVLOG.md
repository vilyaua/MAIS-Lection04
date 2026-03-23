# Development Log

## 2026-03-16

### Initial implementation — Custom ReAct loop refactoring
- Scaffolded `MAIS-Lection04/research-agent/` from lesson-3 base (v0.4.3)
- Replaced all LangChain/LangGraph dependencies with direct OpenAI SDK
- `config.py`: rewrote system prompt with prompt engineering techniques (XML tags, chain-of-thought, few-shot example, negative constraints, structured output template, edge case handling); added `base_url` setting for OpenAI-compatible API flexibility
- `tools.py`: removed `@tool` decorators and `langchain_core` import; kept all 5 tool function bodies unchanged; added `TOOL_SCHEMAS` (OpenAI JSON Schema format) and `TOOL_FUNCTIONS` dispatch dict
- `agent.py`: full rewrite — custom ReAct while-loop replacing `create_react_agent`; two variants: `run_agent_turn()` (returns string) and `run_agent_turn_streaming()` (yields events for SSE); manual tool dispatch, message serialization, iteration limit
- `main.py`: replaced LangChain streaming with `run_agent_turn_streaming()`; kept REPL loop, logging (RotatingFileHandler), token tracking, error handling
- `app.py`: replaced `agent.stream()` with `run_agent_turn_streaming()`; kept sync→async bridge (run_in_executor + Queue), all endpoints, inline HTML/CSS/JS chat UI
- `requirements.txt`: removed `langchain`, `langchain-openai`, `langgraph`; added `openai>=1.70.0`
- Copied as-is: Dockerfile, docker-compose.yml, .env.example, .gitignore, .dockerignore, example_output/report.md
- Created ARCHITECTURE.md reflecting the custom ReAct loop design
- VERSION set to 0.1.0

## 2026-03-19

### 06:00 — Test run: 8 queries from L3

- Ran all 8 test queries from `output/test_queries.txt` against the live server
- Session reset (`POST /api/reset`) between each query for clean token tracking
- All 8 queries produced reports successfully — `write_report` was always called
- Zero retry loops — enhanced prompt's negative constraints prevented failed-call repetition

### 06:35 — L3 vs L4 runtime comparison

- Created `output/L3_vs_L4_comparison.md` — per-query metrics, aggregate stats, report quality analysis
- Parsed both `agent.log` files for token counts, tool calls, errors, timing
- Key findings: L4 used 15% fewer total tokens, 34% fewer output tokens, always saved reports, better error recovery
- L4's enhanced prompt was the biggest driver of quality improvement

### 07:00 — New Session button (`feat/new-session-button`)

- `app.py`: added `POST /api/reset` endpoint — clears `web_messages` and `session_tokens`
- `app.py`: added "New Session" button in sidebar HTML + `resetSession()` JS handler + CSS
- `.gitignore`: removed `logs/` so `agent.log` is tracked in git
- Created PR #1, merged to main

### 07:30 — Documentation

- Created `research-agent/README.md` — setup (Docker + local), usage (web + CLI), architecture, tools, context engineering, system prompt techniques, memory model, L3 vs L4 comparison table
- Updated root `README.md` — project overview, quick start, links to research-agent/README.md
- Created `COMPARISON.md` — detailed file-by-file comparison of every L3→L4 change

## 2026-03-23

### 13:00 — Add truncation to web_search (`feat/web-search-truncation`)

Addressed teacher feedback: "варто додати truncation у web_search, без нього може непередбачувано забиватись контекст"

- `config.py`: added `max_search_content_length: int = 3000` — separate limit from `max_url_content_length` (8000), since search snippets and full page content have different optimal sizes
- `config.py`: added `app_name: str = "Research Agent L04"` for `/api/info` identification
- `tools.py`: added truncation to `web_search` return value using `max_search_content_length`
- `app.py`: added `app` field to `/api/info` endpoint

### 13:25 — Test run: 10 queries with truncation

- Ran all 10 test queries (8 original + 2 RAG-related from L05 set)
- All queries succeeded, total: 265,969 tokens in 544s
- Truncation acts as a safety net — typical search results (~1000 chars) stay under the 3000 limit
- Biggest token consumer: RAG comparison query (65,812 tokens) due to multiple retries on failed Medium URLs
