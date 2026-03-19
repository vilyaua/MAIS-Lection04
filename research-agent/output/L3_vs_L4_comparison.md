# Lesson 3 vs Lesson 4 — Runtime Comparison

> Comparing agent behavior, token usage, tool call patterns, and report quality
> across the same 8 test queries run on both implementations.
>
> - **L3**: LangChain/LangGraph `create_react_agent` (v0.4.3, run 2026-03-15)
> - **L4**: Custom ReAct loop with OpenAI SDK (v0.1.0, run 2026-03-19)
> - **Model**: gpt-4.1-mini (same for both)

---

## 1. Per-Query Metrics

### Q1: "Tell me about the Ukrainian P1-SUN drone"

| Metric | L3 | L4 |
|--------|----|----|
| Total tokens | 15,030 | 18,832 |
| Input tokens | 13,510 | 17,782 |
| Output tokens | 1,520 | 1,050 |
| Tool calls | 7 (web_search:1, read_url:5, write_report:1) | 7 (web_search:2, read_url:4, write_report:1) |
| Fetch errors | 1 (censor.net 403) | 1 (censor.net 403) |
| LLM round-trips | 4 | 6 |
| Duration | ~24s (05:27:52→05:28:16) | ~36s (06:06:29→06:07:05) |

**Notes:** L4 used 2 searches vs L3's 1, finding an additional source about interceptor drones accounting for 30% of air defense kills. L4 report includes more operational statistics (68% success rate, 450 km/h speed upgrade). L4 used more LLM round-trips (6 vs 4) because the custom loop makes individual API calls for each reasoning step, while LangGraph batches some internally.

---

### Q2: "Знайди 3 рецепти здобної паски на Великдень та порівняй їх"

| Metric | L3 | L4 |
|--------|----|----|
| Total tokens | 26,220 | 22,901 |
| Input tokens | 24,047 | 20,883 |
| Output tokens | 2,173 | 2,018 |
| Tool calls | 7 (web_search:3, read_url:3, write_report:1) | 7 (web_search:3, read_url:3, write_report:1) |
| Fetch errors | 0 | 0 |
| LLM round-trips | 4 | 4 |
| Duration | ~37s (10:01:53→10:02:30) | ~39s (06:21:29→06:22:08) |

**Notes:** Nearly identical behavior. L3 used slightly more tokens because the session had accumulated context from earlier queries (MemorySaver). L4 started fresh. Both found 3 recipes from different Ukrainian cooking sites. Report structure and depth are comparable.

---

### Q3: "Compare FastAPI and Streamlit"

| Metric | L3 | L4 |
|--------|----|----|
| Total tokens | 15,968 | 24,077 |
| Input tokens | 14,827 | 22,783 |
| Output tokens | 1,141 | 1,294 |
| Tool calls | 7 (web_search:3, read_url:3, write_report:1) | 11 (web_search:5, read_url:5, write_report:1) |
| Fetch errors | 1 (medium.com 403) | 2 (medium.com 403 x2) |
| LLM round-trips | 4 | 4 |
| Duration | ~24s (12:53:51→12:54:15) | ~44s (06:26:31→06:27:15) |

**Notes:** L4 performed significantly more research — 5 searches and 5 URL reads vs L3's 3+3. The enhanced system prompt's strategy section ("Run 3-5 web searches with varied queries") drove the agent to search for "FastAPI overview", "Streamlit overview", "FastAPI vs Streamlit comparison", "FastAPI features", and "Streamlit features" separately. L3 used fewer, broader searches. This resulted in 50% more tokens but a more thorough report.

---

### Q4: "Compare naive RAG, sentence-window retrieval, and parent-child retrieval"

| Metric | L3 | L4 |
|--------|----|----|
| Total tokens | 40,982 | 47,450 |
| Input tokens | 39,640 | 45,444 |
| Output tokens | 1,342 | 2,006 |
| Tool calls | 8 (web_search:4, read_url:3, write_report:1) | 14 (web_search:6, read_url:7, write_report:1) |
| Fetch errors | 1 (medium.com 403) | 2 (medium.com 403 x2) |
| LLM round-trips | 4 | 6 |
| Duration | ~27s (13:00:15→13:00:42) | ~47s (06:27:41→06:28:28) |

**Notes:** L4 performed nearly double the tool calls (14 vs 8). It searched for each retrieval approach individually + comparison queries, then read 7 URLs including GraphRAG docs, LangChain parent-child chunking guide, and Dify's parent-child retrieval blog. L3 read only 3 URLs. The L4 report is longer (2,006 output tokens vs 1,342) with more detailed per-approach analysis. This is the query where the enhanced prompt's impact is most visible.

---

### Q5: "Compare weather in Costa del Sol and Costa Blanca for March 16-31, 2026"

| Metric | L3 | L4 |
|--------|----|----|
| Total tokens | 37,181 (*) | 32,141 |
| Input tokens | 34,373 | 31,094 |
| Output tokens | 2,808 | 1,047 |
| Tool calls | 9 (web_search:4, read_url:4, write_report:1) | 7 (web_search:3, read_url:3, write_report:1) |
| Fetch errors | 0 | 0 |
| LLM round-trips | 4 | 4 |
| Duration | ~45s (13:50:52→13:51:37) | ~27s (06:28:44→06:29:11) |

(*) L3 actually required 3 separate user messages to complete this query (37,181 + 11,948 + 19,663 = 68,792 total tokens across the multi-turn interaction). The agent initially didn't write a report, requiring follow-up prompts. L4 completed the entire workflow in a single turn.

**Notes:** L4 was more efficient here. The enhanced system prompt's explicit rule "ALWAYS call write_report as your FINAL tool call" prevented the issue L3 had where the agent needed additional prompting to save the report. L4 used fewer tokens and finished faster.

---

### Q6: "Best 3 places in Spain for hip implant surgery"

| Metric | L3 | L4 |
|--------|----|----|
| Total tokens | 14,958 | 29,365 |
| Input tokens | 13,917 | 26,808 |
| Output tokens | 1,041 | 2,557 |
| Tool calls | 7 (web_search:3, read_url:3, write_report:1) | 10 (web_search:5, read_url:4, write_report:1) |
| Fetch errors | 0 | 0 |
| LLM round-trips | 4 | 4 |
| Duration | ~31s (15:26:30→15:27:01) | ~55s (06:29:24→06:30:19) |

**Notes:** L4 did substantially more research (5 searches, 4 URL reads) and produced a longer report (2,557 output tokens vs 1,041). The L4 report includes specific surgeon ratings (4.7/5, 4.4/5), years of experience, cost ranges (€10,000-17,000), and a comparison table. L3's report was more general. The trade-off: 2x token cost for noticeably deeper content.

---

### Q7: "What activities and events are in Velez-Malaga and Torre del Mar in March 2026?"

| Metric | L3 | L4 |
|--------|----|----|
| Total tokens | ~50,471 (*) | 27,507 |
| Input tokens | ~46,500 | 26,194 |
| Output tokens | ~3,971 | 1,313 |
| Tool calls | ~15 (web_search:8, read_url:5, write_report:1, failed:1) | 9 (web_search:4, read_url:4, write_report:1) |
| Fetch errors | 3 (getyourguide, tripadvisor, expedia 403/429) | 0 |
| LLM round-trips | ~10 | 4 |
| Duration | ~70s (04:02:29→04:03:41) | ~46s (06:30:51→06:31:37) |

(*) L3 metrics estimated from raw log; this query predated the "User:" log entries in the current log file.

**Notes:** L3 struggled significantly — hit 3 fetch errors (403/429 from major travel sites) and needed ~10 LLM round-trips to recover. L4 completed cleanly in 4 round-trips with zero errors, finding the Sabor a Málaga gastronomic fair and Torre del Mar events directly. The L4 enhanced prompt's rule "If a tool call fails, adapt — try a different query or skip that source. Do NOT repeat the exact same failed call" helped avoid the retry loops L3 fell into.

---

### Q8: "Compare Vélez-Málaga and Motril for living in 2026"

| Metric | L3 | L4 |
|--------|----|----|
| Total tokens | ~63,010 (*) | 20,656 |
| Input tokens | ~58,000 | 19,424 |
| Output tokens | ~5,010 | 1,232 |
| Tool calls | ~16 (web_search:6, read_url:5+) | 10 (web_search:5, read_url:4, write_report:1) |
| Fetch errors | 1 (idealista.com 403) | 0 |
| LLM round-trips | ~8 | 4 |
| Duration | ~5m32s (03:25:59→03:29:31)(**) | ~41s (06:31:54→06:32:35) |

(*) L3 ran this as the first query of the session (no accumulated context) but still used far more tokens due to LangGraph running 5+ parallel web searches per round and doing extensive multi-round research.
(**) L3 duration includes a long LLM thinking pause.

**Notes:** The largest efficiency gap. L3 ran many parallel web searches per round (LangGraph's behavior) and accumulated massive context. L4's sequential approach with targeted searches was 3x more token-efficient. Both reports cover similar ground (cost of living, property prices, climate, amenities) but L4 is more concise.

---

## 2. Aggregate Comparison

| Metric | L3 (8 queries) | L4 (8 queries) | Delta |
|--------|----------------|-----------------|-------|
| **Total tokens** | ~263,820 | 222,929 | **-15%** |
| **Total input tokens** | ~244,830 | 210,412 | **-14%** |
| **Total output tokens** | ~18,990 | 12,517 | **-34%** |
| **Total tool calls** | ~77 | 75 | ~same |
| **Avg tools/query** | ~9.6 | 9.4 | ~same |
| **Total fetch errors** | 7 | 5 | -29% |
| **Avg LLM round-trips/query** | ~5.3 | 4.5 | -15% |
| **Reports always saved** | No (Q5 needed follow-up) | Yes (all 8) | improved |

### Token Efficiency
- L4 used **15% fewer total tokens** across all 8 queries
- L4's biggest savings were on queries where L3 struggled (Q7: -45%, Q8: -67%)
- L4 used more tokens than L3 on some queries (Q3, Q4, Q6) because the enhanced prompt drove deeper research
- L4's output tokens were 34% lower — more concise reports

### Tool Call Patterns
- L3 averaged 3.1 searches + 3.4 reads per query
- L4 averaged 4.1 searches + 4.3 reads per query
- L4 consistently did more research but was more targeted (fewer wasted calls)
- Both averaged 1 write_report per query

### Error Handling
- L3 hit 7 fetch errors, some causing retry loops (Q7 had 3 errors → 10 LLM round-trips)
- L4 hit 5 fetch errors but recovered gracefully in all cases (never exceeded 6 round-trips)
- L4's negative constraint "Do NOT repeat the exact same failed call" prevented retry spirals

---

## 3. Report Quality Comparison

### Structure
- **L3**: Reports follow basic Markdown with H1/H2 headers, sourced from the basic system prompt
- **L4**: Reports follow the XML-tagged `<output_format>` template more consistently: Title → Introduction → Sections → Comparison → Conclusion → Sources

### Depth
- **L4 generally deeper**: More specific data points, ratings, and statistics (e.g., Q6 hip surgery includes surgeon experience years, patient ratings, cost ranges)
- **L3 occasionally broader**: L3's Velez comparison (Q8) covered more sub-topics due to massive parallel searches

### Language Handling
- Both handled Ukrainian queries well (Q2 in Ukrainian throughout)
- L4's Ukrainian report was more structured with a clear comparison table

### Sources
- L3: 2-4 sources per report
- L4: 3-5 sources per report
- L4 tended to cite more diverse sources

---

## 4. Behavioral Differences

### Memory Model
| Aspect | L3 (MemorySaver) | L4 (messages list + reset) |
|--------|-------------------|----------------------------|
| Cross-query context | Full history accumulates | Fresh per session (reset between queries) |
| Token growth | Later queries cost more (full history re-sent) | Flat per query |
| Follow-up capability | Automatic (same thread) | Manual (messages list persists within session) |

L3's MemorySaver caused input tokens to grow as the session progressed (Q1 at 653 input tokens → Q18 at 13,917 input tokens for comparable complexity). L4's per-query resets kept costs predictable.

### Search Strategy
- **L3**: LangGraph sometimes dispatched multiple parallel tool calls in a single step (visible in logs as multiple `response:` lines at the same timestamp)
- **L4**: Sequential tool execution — one search → process results → decide next action. More predictable but potentially slower.

### Report Saving
- **L3**: Occasionally forgot to call `write_report` (Q5 weather needed 3 user turns)
- **L4**: Never forgot — the enhanced prompt's explicit rule + few-shot example (showing write_report as final step) eliminated this issue

### Error Recovery
- **L3**: Sometimes retried the same failing URL or entered long search loops (Q7: 10 LLM round-trips)
- **L4**: Adapted quickly — skipped failing sources and tried alternatives (max 6 round-trips)

---

## 5. System Prompt Impact

The L4 enhanced prompt (120 lines with XML tags) vs L3 basic prompt (69 lines flat Markdown) produced these measurable differences:

| Technique | Observable Effect |
|-----------|-------------------|
| `<strategy>` with step-by-step | L4 consistently searched 3-5 times (matching the prompt's guidance) |
| `<example>` few-shot | L4 always followed the search→read→write_report pattern |
| `<rules>` negative constraints | L4 never repeated failed URLs, never hallucinated URLs |
| `<edge_cases>` | Not triggered in these tests (no vague queries or empty results) |
| `<output_format>` template | L4 reports have more consistent structure |

---

## 6. Architecture Trade-offs

| Dimension | L3 (LangChain) | L4 (Custom) |
|-----------|----------------|-------------|
| **Lines of code** (agent.py) | 21 | 244 |
| **Dependencies** | 3 framework packages | 1 SDK package |
| **Debugging** | Opaque (framework internals) | Transparent (every step visible in code) |
| **Flexibility** | Limited by framework API | Full control over loop, serialization, streaming |
| **Parallel tool calls** | Automatic (LangGraph feature) | Not implemented (sequential only) |
| **Token efficiency** | -15% (higher due to framework overhead + memory growth) | Baseline |
| **Error resilience** | Framework catches some errors | All error paths are explicit |
| **Streaming** | Chunk-based (agent/tools dicts) | Event-based (typed dicts) |

---

## 7. Conclusion

**L4 (custom ReAct loop) advantages:**
- 15% fewer total tokens across the same 8 queries
- More predictable behavior — always saves reports, recovers from errors gracefully
- Enhanced prompt produces deeper, more structured research
- Transparent debugging — every LLM call and tool dispatch is visible in code
- No framework dependency overhead

**L3 (LangChain/LangGraph) advantages:**
- 10x less code to write and maintain
- Automatic parallel tool calls (can be faster for independent searches)
- Built-in memory management with MemorySaver
- Easier to add new agent patterns (tool routing, sub-agents)

**Bottom line:** The custom loop achieved better results with less cost, primarily because the enhanced system prompt guided the agent more effectively — not because of the loop implementation itself. The prompt engineering was the biggest driver of quality improvement.
