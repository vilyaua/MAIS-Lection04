# MAIS-Lection04

## Homework: Research Agent v2 (Custom ReAct Loop)

Refactored Research Agent from Lesson 3 — replaces all LangChain/LangGraph abstractions with a hand-rolled ReAct loop and enhanced system prompt, using the OpenAI SDK directly.

### Project Structure

- **`homework-lesson-4/`** — Original homework skeleton with task description
- **`research-agent/`** — Completed implementation
- **`AGENTS.md`** — Implementation guide and specifications
- **`COMPARISON.md`** — Detailed file-by-file comparison of L3 vs L4 changes

### Quick Start

```bash
cd research-agent
pip install -r requirements.txt
cp .env.example .env   # add your API key
python main.py
```

See [`research-agent/README.md`](research-agent/README.md) for full setup and architecture details.
