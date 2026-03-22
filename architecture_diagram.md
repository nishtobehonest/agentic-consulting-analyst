# Agentic Consulting Analyst — Architecture Diagram

## System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USER INTERFACES                               │
│   Web Dashboard (React + Vite)  │  CLI  │  Future: Slack Bot        │
└──────────────────────┬──────────────────────────────────────────────┘
                       │ HTTP / SSE stream
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATION LAYER                               │
│                  (Vercel Serverless / api/chat.js)                   │
│                                                                      │
│  1. Intent Classification → simple lookup OR complex synthesis?      │
│  2. Model Routing       → Haiku (60%) OR Sonnet (40%)                │
│  3. Tool Planning       → which tools are needed for this query?     │
│  4. Parallel Tool Calls → fire tools concurrently                    │
│  5. Synthesis           → combine tool results + apply framework     │
│  6. Citation Injection  → tag every factual claim with source        │
│  7. Stream Response     → SSE back to UI (show intermediate steps)   │
│  8. Eval Logging        → async log to evaluation pipeline           │
└──────┬──────────┬───────────┬──────────┬──────────┬─────────────────┘
       │          │           │          │          │
       ▼          ▼           ▼          ▼          ▼
  ┌────────┐ ┌────────┐ ┌─────────┐ ┌───────┐ ┌──────────┐
  │  RAG   │ │Finance │ │  News   │ │ Calc  │ │  Chart   │
  │  Tool  │ │  Tool  │ │  Tool   │ │ Tool  │ │   Tool   │
  └───┬────┘ └───┬────┘ └────┬────┘ └───────┘ └──────────┘
      │          │            │
      ▼          ▼            ▼
  Pinecone   Yahoo Finance  Bing News
  Vector DB  / Alpha Vantage  Search API
```

---

## Context Window Composition (Per Request)

```
┌─────────────────────────────────────────────────────────┐
│                   CONTEXT WINDOW                         │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  SYSTEM PROMPT (~1,000 tokens)         CACHED ✓  │   │
│  │  Role + scope rules + output format              │    │
│  │  Active case name + available frameworks         │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  RAG CHUNKS (~1,500 tokens)          Layer 1+2   │   │
│  │  Top-5 relevant passages from case PDFs          │    │
│  │  Structured financial data for active case       │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  USER MEMORY SUMMARY (~200 tokens)   Layer 4     │   │
│  │  "Practiced Porter's 5x, weak on DCF"            │    │
│  │  "Prefers BCG matrix framing"                    │    │
│  │  "Don't re-explain basic frameworks"             │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  CONVERSATION HISTORY (~800 tokens)  Layer 5     │   │
│  │  Last 8 turns — sliding window                   │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  ACTIVE TAB (~100 tokens)            Existing ✓  │   │
│  │  "User is on the Financials tab"                 │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  Total: ~3,600 tokens   (vs ~8,000+ current system)     │
└─────────────────────────────────────────────────────────┘
```

---

## Memory Architecture (3 Types)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MEMORY SYSTEM                                │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  SEMANTIC MEMORY (RAG)                                        │   │
│  │  Pinecone Vector DB                                           │   │
│  │                                                               │   │
│  │  Index 1: Case PDFs (chunked ~500 tokens each)                │   │
│  │    - oracle-vs-salesforce.pdf      → ~200 chunks              │   │
│  │    - dominos-pizza.pdf             → ~150 chunks              │   │
│  │    - apple-augmented-reality.pdf   → ~180 chunks              │   │
│  │    - upwork-human-cloud.pdf        → ~160 chunks              │   │
│  │    - epic-health-it.pdf            → ~170 chunks              │   │
│  │                                                               │   │
│  │  Index 2: Structured data (caseData.js → embedded as text)   │   │
│  │    - Financial time series, market share, acquisition data    │   │
│  │                                                               │   │
│  │  Retrieved: top-5 chunks per query via cosine similarity      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  EPISODIC MEMORY (Per-User History)                           │   │
│  │  Supabase (free tier)                                         │   │
│  │                                                               │   │
│  │  conversations table:                                         │   │
│  │    user_id, session_id, turn, role, content, timestamp        │   │
│  │                                                               │   │
│  │  user_profiles table:                                         │   │
│  │    user_id, frameworks_practiced[], weak_spots[],             │   │
│  │    preferred_style, corrections_made[]                        │   │
│  │                                                               │   │
│  │  Injected as: 3-5 bullet summary (LLM-compressed)            │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  PROCEDURAL MEMORY (Skills / Frameworks)                      │   │
│  │  Baked into system prompt as selectable templates             │   │
│  │                                                               │   │
│  │  Available skills (selected by intent classifier):            │   │
│  │    - porter_five_forces_template                              │   │
│  │    - swot_analysis_template                                   │   │
│  │    - bcg_matrix_template                                      │   │
│  │    - dcf_skeleton_template                                    │   │
│  │    - competitive_moat_framework                               │   │
│  │    - case_interview_structure (situation/complication/Q)      │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Tool Definitions

```
┌─────────────────────────────────────────────────────────────────────┐
│                           TOOLS                                      │
│                                                                      │
│  retrieve_case_chunks(query: string, case?: string, top_k?: int)    │
│    → chunks: [{text, source, page, score}]                          │
│    Pinecone semantic search over all 5 case PDFs                    │
│    Used: always (every query)                                       │
│                                                                      │
│  get_financial_data(company: string, metric: string,                │
│                     start: date, end: date)                         │
│    → {values: [{date, value}], unit: string}                        │
│    Yahoo Finance + Alpha Vantage APIs                               │
│    Used: when query needs current or post-case data                 │
│                                                                      │
│  search_recent_news(query: string, days_back?: int)                 │
│    → articles: [{headline, source, date, url}]                      │
│    Bing News Search API                                             │
│    Used: when query asks about recent developments                  │
│                                                                      │
│  calculate_ratio(numerator: number, denominator: number,            │
│                  label?: string)                                     │
│    → {result: number, label: string}                                │
│    Safe server-side arithmetic (prevents hallucinated math)         │
│    Used: for any financial ratio/calculation                        │
│                                                                      │
│  generate_chart(data: object[], type: "bar"|"line"|"pie",           │
│                 title: string, axes: {x, y})                        │
│    → chart_component_json (rendered by Recharts on frontend)        │
│    Used: when visual comparison would clarify the analysis          │
│                                                                      │
│  save_insight(user_id: string, title: string, content: string,      │
│               tags: string[])                                        │
│    → {saved: true, insight_id: string}                              │
│    Writes to user's personal analysis notebook in Supabase          │
│    Used: when user asks to save, bookmark, or note something        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## MCP Server Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        MCP SERVERS                                   │
│                                                                      │
│  financial-data-mcp                                                  │
│    Exposes: get_stock_price, get_revenue_data, get_market_cap        │
│    Wraps: Yahoo Finance API + Alpha Vantage API                     │
│    Auth: API keys in environment variables                           │
│                                                                      │
│  web-search-mcp                                                      │
│    Exposes: search_web, search_news, get_page_content                │
│    Wraps: Bing Search API (news) + Brave Search (general)            │
│    Rate limit: 3 req/sec (Bing free tier)                           │
│                                                                      │
│  case-library-mcp                                                    │
│    Exposes: retrieve_chunks, list_cases, get_case_metadata           │
│    Wraps: Pinecone retrieval + case metadata index                   │
│    Auth: Pinecone API key                                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6-Layer Context Map (OpenAI Parallel)

```
OpenAI Data Agent              Agentic Consulting Analyst
─────────────────────────────────────────────────────────
Layer 1: Schema metadata   →  Case data schema: companies,
                              metrics, time periods, financials
                              (from caseData.js structure)

Layer 2: Expert table      →  Professor-annotated case summaries,
         descriptions         applicable frameworks per case,
                              key analytical angles

Layer 3: Institutional     →  Cornell DTI course materials,
         knowledge            assignment rubrics,
         (Slack/Notion)        prior class discussion notes

Layer 4: Learning memory   →  Per-user correction memory:
         (prior corrections)   "Already knows Porter's 5x,
                               push to advanced application"

Layer 5: Historical query  →  Most-common student questions →
         patterns              prioritized in retrieval ranking;
                               canonical analyses surfaced first

Layer 6: Fallback live     →  Real-time financial + news APIs
         warehouse queries     when case data is insufficient
```

---

## What's Already Built (No Re-implementation Needed)

```
EXISTING CODE TO EXTEND (not replace):

dashboard/src/utils/caseContext.js
  Current: serializes ALL case data → single system prompt (noisy)
  Extend:  inject RAG chunks + user memory summary instead
  Change:  ~50 lines

dashboard/api/chat.js  (and server.js for local dev)
  Current: single Gemini call with system prompt
  Extend:  add tool-calling loop around existing SSE stream
  Change:  ~80 lines

dashboard/src/data/caseData.js
  Current: structured financial data for Oracle vs Salesforce
  Role in agent: Layer 1 structured data source, embedded into Pinecone
  Change:  none (becomes one input to embedding pipeline)

5 PDF files in repo root:
  Role in agent: primary knowledge base → chunk → embed → Pinecone
  Change:  none (run one-time embedding script)
```

---

## Data Flow: Example Query

```
User types: "How has Oracle's cloud strategy evolved, and is it working today?"
                                   │
                    ┌──────────────▼─────────────┐
                    │    Intent Classification    │
                    │    → complex synthesis      │
                    │    → route to Sonnet        │
                    └──────────────┬─────────────┘
                                   │
                    ┌──────────────▼─────────────┐
                    │       Tool Planning         │
                    │  [RAG, financial, news]     │
                    └──────────────┬─────────────┘
                                   │
              ┌────────────────────┼───────────────────────┐
              ▼                    ▼                        ▼
     RAG: "Oracle cloud       Financial API:          News: "Oracle
     strategy" → 5 chunks     Oracle cloud revenue    cloud 2024 2025"
     from oracle-vs-          % 2020-2025             → 4 headlines
     salesforce.pdf
              │                    │                        │
              └────────────────────┼───────────────────────┘
                                   ▼
                    ┌──────────────────────────┐
                    │  Synthesis + BCG Matrix  │
                    │  Skills template applied │
                    └──────────────┬───────────┘
                                   │
                    ┌──────────────▼───────────┐
                    │  Stream response with    │
                    │  citations:              │
                    │  "[Oracle case, p.7]..." │
                    └──────────────────────────┘
```
