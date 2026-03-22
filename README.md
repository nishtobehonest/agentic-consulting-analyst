# Agentic Consulting Analyst

> The next phase of the [Case Study Dashboard](https://github.com/nishtobehonest/oracle-salesforce-dashboard) — evolving from a static data explorer into an autonomous business intelligence agent.

---

## What This Is

The current dashboard answers questions about case data. This agent does something harder: it reasons *across* multiple cases, remembers what you already know, pulls live market data, and cites every claim it makes.

The gap between the two is the story this document tells.

---

## The Problem It Solves

| Pain point | Current dashboard | This agent |
|---|---|---|
| Cross-case synthesis | One case at a time | Reasons across all 5 cases simultaneously |
| Stale data | Case PDF from 2021 | Live financial + news APIs |
| Personalization | No memory | Tracks your gaps, frameworks practiced, preferred style |
| Source grounding | All data in system prompt (noisy) | RAG: top-5 relevant chunks per query |
| Multi-step reasoning | Single LLM call | Intent → tool planning → parallel execution → synthesis |

**Market context:** ~50,000 students apply to top consulting firms annually. Existing tools (RocketBlocks $60–90/month, human coaches $200+/hr) don't work from real, data-rich case studies with live market grounding. This does.

---

## Architecture

### Orchestration Layer

Every query flows through 8 steps:

```
1. Intent Classification   → simple lookup OR complex synthesis?
2. Model Routing           → Haiku (60%) OR Sonnet (40%)
3. Tool Planning           → which tools are needed?
4. Parallel Tool Calls     → fire tools concurrently
5. Synthesis               → combine results + apply framework
6. Citation Injection      → tag every factual claim with source
7. Stream Response         → SSE back to UI (show intermediate steps)
8. Eval Logging            → async log to evaluation pipeline
```

### Tools

| Tool | Does | Used when |
|---|---|---|
| `retrieve_case_chunks(query)` | RAG over all 5 case PDFs via Pinecone | Every query |
| `get_financial_data(company, metric, range)` | Yahoo Finance + Alpha Vantage | Query needs current/post-case data |
| `search_recent_news(query)` | Bing News Search API | User asks about recent developments |
| `calculate_ratio(a, b)` | Safe server-side arithmetic | Any financial ratio/calculation |
| `generate_chart(data, type)` | New Recharts visualizations on demand | Visual comparison would help |
| `save_insight(user_id, text)` | Write to user's analysis notebook (Supabase) | User asks to save/bookmark |

### Context Window (Per Request)

```
System prompt          ~1,000 tokens   CACHED — same every request
Retrieved case chunks  ~1,500 tokens   RAG from Pinecone (top-5 passages)
User memory summary      ~200 tokens   Supabase: frameworks practiced, weak spots
Conversation history     ~800 tokens   Sliding window: last 8 turns
Active tab context        ~100 tokens   Already exists in caseContext.js
─────────────────────────────────────
Total:                 ~3,600 tokens   (vs. ~8,000+ current dump-everything approach)
```

### Memory System

| Type | Stores | Implementation |
|---|---|---|
| **Semantic (RAG)** | All 5 case PDFs + structured financial data | Pinecone vector DB |
| **Episodic** | Per-user conversation history, learning profile | Supabase (free tier) |
| **Procedural** | Framework templates: Porter's 5, SWOT, BCG, DCF | System prompt, selected by intent classifier |

---

## What's Already Built

The agent extends the existing dashboard — it doesn't replace it.

| File | Current role | Change needed |
|---|---|---|
| `src/utils/caseContext.js` | Serializes all case data into system prompt | Inject RAG chunks + user memory summary instead (~50 lines) |
| `api/chat.js` + `server.js` | Single Gemini call with system prompt | Add tool-calling loop around existing SSE stream (~80 lines) |
| `src/data/caseData.js` | Structured Oracle/Salesforce financials | Becomes Layer 1 input to Pinecone embedding pipeline (no changes) |
| `*.pdf` (5 case PDFs) | Source material | Run one-time chunking + embedding script → Pinecone |

---

## Evaluation Framework

Six automated layers run on every git push. No pilot group, no manual review — the pipeline gates deploys.

| Layer | What it tests | Target |
|---|---|---|
| **1. Golden Answer Benchmarks** | 150 curated Q&A pairs (factual + analytical) | >95% factual accuracy, >80% analytical similarity |
| **2. RAG Retrieval Quality** | 50 questions with known source passage locations | Recall@5 >85%, Precision@1 >65% |
| **3. LLM-as-Judge** | Claude Opus scores random 5% of live queries on 4 dimensions | Avg >4.0/5.0 across all dimensions |
| **4. Consistency Testing** | Same question × 10 runs; measure answer variance | <15% variance on factual, <30% on analytical |
| **5. Hallucination Trap Detection** | 20 false-premise questions; agent must correct, not accept | >95% correction rate |
| **6. Scope Boundary Testing** | Off-topic, ambiguous, and out-of-knowledge queries | >98% off-topic decline, >90% "I don't know" rate |

**Eval runtime: ~8 minutes per deploy.** All gates must be green before production.

The analytical scoring (Layer 1) uses cosine similarity on embeddings — same approach as OpenAI's internal data agent. Correctness, not exact wording.

---

## Cost Model (100 Active Users)

Assumes 3 sessions/week × 15 queries/session × 4.3 weeks = ~19,350 queries/month.

| Item | Monthly cost |
|---|---|
| LLM API — 60% Haiku / 40% Sonnet + prompt caching | $171 |
| Pinecone Starter (5 cases, ~50K chunks) | $70 |
| Text embedding API | $5 |
| Bing News Search API | $25 |
| Supabase (user memory) | $0 (free tier) |
| Vercel Pro | $20 |
| **Total** | **~$291/month** |

**Per-user cost: ~$2.91/month.** At $20/month subscription: **85% gross margin**, break-even at 15 users.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite, Tailwind CSS, Recharts (existing) |
| Orchestration | Vercel serverless (`api/chat.js`) |
| Primary AI | Claude Sonnet (complex) + Claude Haiku (simple) |
| RAG | Pinecone vector DB |
| User memory | Supabase |
| Financial data | Yahoo Finance API + Alpha Vantage |
| News | Bing News Search API |
| Evaluation | Python + pytest + OpenAI embeddings (CI/CD) |

---

## Documents in This Folder

| File | Contents |
|---|---|
| `architecture_diagram.md` | Full system design with ASCII diagrams: orchestration, context window, memory, tools, MCP servers, and a worked example query |
| `presentation_outline.md` | 15-min video presentation script for Cornell DTI AI for Business Applications course — includes business case, evaluation strategy, cost model, and speaker notes |
| `evaluation_framework.md` | Full specification of the 6-layer automated evaluation pipeline with code pseudocode, metrics targets, and CI/CD architecture |

---

## Context

Companion to the [live Oracle vs Salesforce dashboard](https://oracle-salesforce-dashboard.vercel.app), built during the **Digital Technology Innovation** course at Cornell (MEM, Class of 2026). Architecture is directly mapped from OpenAI's published in-house data agent framework — each layer has an explicit parallel.

The current dashboard has a file in its repo called `ToDos` with one item: *"Make this a RAG system."* This is that.
