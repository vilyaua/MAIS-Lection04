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
