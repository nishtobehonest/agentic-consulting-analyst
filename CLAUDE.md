# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Architecture and presentation documentation for the **Agentic Consulting Analyst** — the next phase of the [Oracle vs Salesforce dashboard](https://oracle-salesforce-dashboard.vercel.app). No runnable code lives here; all implementation targets the companion `dashboard/` repo.

## Documents

| File | Purpose |
|---|---|
| `README.md` | Project overview, problem framing, cost model, tech stack |
| `architecture_diagram.md` | Full system design with ASCII diagrams: 8-step orchestration flow, context window composition, 3-type memory system, tool definitions, MCP servers, worked example query |
| `evaluation_framework.md` | 6-layer automated eval pipeline: golden benchmarks, RAG retrieval quality, LLM-as-judge, consistency testing, hallucination trap detection, scope boundary testing |
| `presentation_outline.md` | 15-min video script for Cornell DTI AI for Business Applications course — includes slide-by-slide notes, cost/revenue breakdown, and key talking points |

## Architecture in Brief

### Orchestration (8 steps per query)
Intent classification → model routing (Haiku 60% / Sonnet 40%) → tool planning → parallel tool calls → synthesis → citation injection → SSE stream → eval logging

### Context Window (~3,600 tokens total)
- System prompt (~1,000 tokens, cached)
- RAG chunks from Pinecone (~1,500 tokens, top-5 passages)
- User memory summary from Supabase (~200 tokens)
- Conversation history sliding window (~800 tokens, last 8 turns)
- Active tab context (~100 tokens, existing in `caseContext.js`)

### Memory (3 types)
- **Semantic**: Pinecone — 5 case PDFs chunked ~500 tokens each (~860 total chunks)
- **Episodic**: Supabase — per-user `conversations` and `user_profiles` tables
- **Procedural**: System prompt templates — Porter's 5, SWOT, BCG, DCF, selected by intent classifier

### Tools
`retrieve_case_chunks`, `get_financial_data` (Yahoo Finance + Alpha Vantage), `search_recent_news` (Bing), `calculate_ratio`, `generate_chart` (Recharts), `save_insight` (Supabase)

### MCP Servers
`financial-data-mcp`, `web-search-mcp`, `case-library-mcp`

## What's Already Built in the Dashboard Repo

These files need extension, not replacement:

| File | Required change |
|---|---|
| `src/utils/caseContext.js` | Replace full-data dump with RAG chunks + user memory injection (~50 lines) |
| `api/chat.js` + `server.js` | Wrap existing SSE stream in a tool-calling loop (~80 lines each) |
| `src/data/caseData.js` | No changes — becomes Pinecone embedding input |
| `*.pdf` (5 cases) | No changes — run one-time chunking + embedding script |

## Evaluation Pipeline

Six automated layers gate every deploy. Target runtime: ~8 minutes.

| Layer | What it tests | Key target |
|---|---|---|
| 1. Golden Benchmarks | 150 Q&A pairs (factual + analytical) | >95% factual, >80% analytical at 0.75 cosine |
| 2. RAG Retrieval | 50 questions with known source passages | Recall@5 >85%, Precision@1 >65% |
| 3. LLM-as-Judge | Claude Opus scores 5% sample on 4 dimensions | Avg >4.0/5.0 |
| 4. Consistency | 30 questions × 10 runs, measure variance | <15% factual, <30% analytical |
| 5. Hallucination Traps | 20 false-premise questions | >95% correction rate |
| 6. Scope Boundary | Off-topic, ambiguous, out-of-knowledge queries | >98% off-topic decline |

Layer 3 runs async on live traffic (too expensive per push). Layers 1, 2, 4, 5, 6 are CI/CD gates.

## Cost Model (100 Users)

~19,350 queries/month → **~$291/month total** → **$2.91/user**. Break-even at 15 users at $20/month subscription (85% gross margin).
