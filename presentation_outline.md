# Agentic Consulting Analyst — Presentation Outline
## AI for Business Applications | Cornell MEM

**Format:** 10-15 min video presentation
**Total points:** 15 pts

---

## Slide 1: Title

**Title:** Agentic Consulting Analyst: A 24/7 AI That Knows Every Case Cold
**Subtitle:** From Static Dashboards to an Autonomous Business Intelligence Agent

**Opening hook (spoken):**
> "This is a live product I already built — an interactive case study dashboard for Oracle vs Salesforce, deployed on Vercel. It can answer questions about the data you're looking at. But it can't reason across multiple cases, it doesn't know who you are, it can't pull today's market data, and every conversation starts from scratch. That's the gap this agent fills."

*[Show live dashboard at oracle-salesforce-dashboard.vercel.app]*

---

## Slides 2-3: The Business & AI Problem [2.5 pts]

### The Business Problem

**Who:** MEM/MBA students globally preparing for McKinsey, Bain, BCG case interviews — and the Cornell DTI students using this dashboard today.

**The core pain:**
- Case interview prep is expensive and scarce: $200-500/hr for a human coach, limited availability, and they forget what you already know
- Students have rich case study PDFs (Oracle/Salesforce, Domino's, Apple AR, Upwork, Epic) but no way to *interact* with them — static reading only
- Cases go stale: the PDF was written in 2021 and has no idea what happened to Salesforce in 2025
- No cross-case synthesis: "How does Oracle's cloud transition compare to Domino's digital turnaround?" requires someone who has read *all five cases* — no human TA can do that at scale for 200 students
- Professors can't give every student personalized analytical feedback

**The market context:**
- ~50,000 students apply to top consulting firms annually (global)
- Existing tools: RocketBlocks ($60-90/month, generic frameworks), CaseCoach ($200+/session, human), CasePrepared ($30/month, pre-recorded)
- None of them work from *real, data-rich case studies with live market grounding*

**Who benefits:**
- Primary: Consulting candidates — unlimited practice sessions with real cases + structured feedback
- Secondary: Cornell DTI students — post-class exploration without TA bottleneck
- Future: Consulting firms onboarding analysts to internal proprietary case libraries

---

### The AI Problem

**Why AI?** This problem requires:
1. **Synthesis at scale** — unstructured PDFs + structured financials + live market signals, simultaneously
2. **Multi-step reasoning** — retrieve relevant case passage → apply Porter's Five Forces → cross-reference current market data → generate insight → cite sources. Can't be done in a single prompt.
3. **Personalization** — remembers what frameworks you've already drilled, what mistakes you keep repeating, where your gaps are
4. **Real-time grounding** — a Salesforce case needs to know Salesforce's current market cap and recent news, not just 2021 data

**Why can't you just ask ChatGPT?**

| What you need | What ChatGPT gives you |
|---|---|
| Your specific case PDFs (institutional knowledge) | Its training data — no access to your files |
| Memory of your learning journey across sessions | Forgets everything between conversations |
| Real-time financial data (today's revenue, market cap) | Knowledge cutoff, no live data tools |
| Multi-step orchestration (retrieve → analyze → cite) | Single-turn response, no chained tool use |
| Curriculum alignment (your professor's frameworks) | Generic answers with no course context |
| Structured deliverables (comparison tables, scorecards) | Freeform text — you reformat it manually |

**Data needed (all feasible to acquire):**
- ✅ Case study PDFs — already have 5 in the repo (Oracle/Salesforce, Apple AR, Domino's, Epic, Upwork)
- ✅ Structured financial data — already in `caseData.js`, expandable to all cases
- ✅ Real-time market data — Yahoo Finance API (free), Alpha Vantage (free tier)
- ✅ Recent news — Bing News Search API (free tier: 3 calls/sec)
- ✅ User interaction data — collected from live chat sessions for memory/personalization

---

## Slides 4-5: Evaluation Strategy [5 pts]

**The core challenge:** Can't use a pilot group — that doesn't scale. We need automated evaluation that runs before any real user ever touches the system.

**Key insight from OpenAI's in-house data agent:**
> "The only way to scale capability without breaking trust is through systematic evaluation."
Their method: curated Q&A pairs with "golden answers," compared programmatically (not string matching — semantic equivalence). We adapt this directly to case analysis.

---

### 6-Layer Automated Evaluation Framework

**Layer 1: Golden Answer Benchmarks**
- Curate 150 Q&A pairs with verified answers from case PDFs
- Two types:
  - *Factual*: "What was Salesforce's revenue CAGR 2015-2020?" → exact number from `caseData.js`
  - *Analytical*: "Apply Porter's Five Forces to CRM market circa 2015" → professor-authored gold answer, scored by semantic similarity (cosine similarity of embeddings, not string match)
- Run on every git push via CI/CD pipeline
- Track accuracy curve over time → catch regressions before deploy

**Layer 2: RAG Retrieval Quality**
- Build retrieval test set: 50 questions with known source passage locations in PDFs
- Metrics: **recall@5** (right chunk in top 5?) and **precision@1** (right chunk ranked first?)
- Automated — zero humans, just vector similarity scoring
- Directly analogous to OpenAI's "compare generated SQL to golden SQL"

**Layer 3: LLM-as-Judge**
- Use Claude Opus as independent evaluator (separate model from the agent)
- Score 5% random sample of live queries on 4 dimensions: (a) factual accuracy vs case data, (b) citation quality, (c) analytical depth, (d) pedagogical clarity
- Returns structured JSON → aggregated into a metrics dashboard
- No human needed; Opus can assess analytical quality at scale

**Layer 4: Consistency / Stability Testing**
- Same question × 10 runs → measure semantic variance in answers
- High variance = hallucination risk → auto-flagged for inspection
- Targets: <15% variance on factual questions, <30% on analytical questions

**Layer 5: Hallucination Trap Detection**
- Inject false premises into test set: "Given Oracle's revenue declined 30% in 2022, explain why..."
- Agent must *correct* the false premise, not accept it
- 20 trap questions automated; target: >95% refusal/correction rate

**Layer 6: Scope Boundary Testing**
- From OpenAI: "Reliability degrades quickly when scope expands faster than governance"
- Send off-topic queries (personal advice, non-case questions) → measure graceful decline rate
- Send ambiguous queries → measure "I'm not certain" rate vs hallucination rate
- All automated in CI/CD

**Rollout gate:** All 6 layers must be green on staging before production deploy. No manual review needed — the pipeline enforces it.

**Visual for slide:** Show a simple CI/CD pipeline diagram: code push → 6-layer eval → green/red gate → deploy/block.

---

## Slides 6-8: Agent Architecture [5 pts]

**Core metaphor:**
> "Imagine a junior consultant who has read every case file cover-to-cover, has a live Bloomberg terminal, and never forgets a single conversation with any client. That's what this agent is."

---

### The 6-Layer Context Architecture
*(Directly mapped from OpenAI's in-house data agent framework)*

| Layer | OpenAI's System | Our Agentic Consulting Analyst |
|---|---|---|
| **1** | Schema metadata | Case data schema: companies, metrics, time periods, financial structure |
| **2** | Expert descriptions of key tables | Professor-annotated case summaries + applicable frameworks per case |
| **3** | Institutional knowledge (Slack, Notion) | Cornell DTI course materials, assignment rubrics, prior class discussion notes |
| **4** | Learning memory (prior corrections) | Per-user memory: "Already knows Porter's Five Forces, push to advanced application" |
| **5** | Historical query patterns | Most common student questions → prioritized in retrieval ranking |
| **6** | Fallback live queries | Real-time financial/news API when case data is insufficient |

**Key OpenAI insight applied directly:**
> "Context curation matters more than volume — carefully curated, accurate context outperformed large, noisy datasets."

This is *exactly* why RAG beats the current system. Right now, `caseContext.js` serializes ALL financial data into every prompt — noisy and wasteful. The agentic system retrieves only the 3-5 most relevant chunks per query. Less context, higher accuracy.

---

### What's In the Context Window (Per Request)

```
System prompt (~1,000 tokens)          ← CACHED (same every request → 90% cost savings on this portion)
  Role, scope rules, output format, active case name

Retrieved case chunks (~1,500 tokens)  ← RAG from Pinecone over 5 case PDFs
  Top-5 most relevant passages to the query

User memory summary (~200 tokens)      ← From Supabase (episodic memory)
  "Has practiced Porter's 5x, weak on financial analysis, prefers BCG matrix"

Conversation history (~800 tokens)     ← Sliding window: last 8 turns
  Maintains analytical thread without unbounded growth

Active tab context (~100 tokens)       ← Already implemented in caseContext.js
  "User is currently on the Financials tab"
```

Total: ~3,600 tokens per query (vs. ~8,000+ in the current dump-everything approach)

---

### Memory Architecture

| Type | What it stores | Implementation |
|---|---|---|
| **Semantic (RAG)** | Case PDF content + structured financial data | Pinecone vector DB — retrieved per query |
| **Episodic** | Per-user conversation history, learning profile | Supabase (free tier) — summarized on inject |
| **Procedural (Skills)** | Pre-built frameworks: Porter's 5, SWOT, BCG Matrix, DCF skeleton | System prompt templates — selected by intent |

---

### Tools

| Tool | What it does | Why it matters |
|---|---|---|
| `retrieve_case_chunks(query)` | RAG over all 5 case PDFs | Grounds answers in actual case content |
| `get_financial_data(company, metric, range)` | Alpha Vantage / Yahoo Finance | Makes stale cases current |
| `search_recent_news(query)` | Bing News API | "What happened to Salesforce after this case was written?" |
| `calculate_ratio(a, b)` | Safe arithmetic | Prevents hallucinated financial calculations |
| `generate_chart(data, type)` | New Recharts visualizations on demand | Visual output, not just text |
| `save_insight(user_id, text)` | Write to user's analysis notebook | Persistent artifact creation |

**MCP Servers:**
- Financial data MCP (wraps Yahoo Finance + Alpha Vantage)
- Web search MCP (Bing/Brave)
- Case library MCP (Pinecone retrieval interface)

---

### Orchestration Flow

```
User: "How does Oracle's cloud strategy compare to Salesforce's, and is it still relevant today?"

1. Intent classification → complex synthesis (route to Sonnet, not Haiku)
2. Tool plan → [retrieve_case_chunks, get_financial_data, search_recent_news]
3. Parallel tool calls:
   ├── RAG: retrieve Oracle + Salesforce cloud strategy passages
   ├── Financial API: pull current cloud revenue % for both companies
   └── News: search "Oracle cloud 2024 2025"
4. Synthesis: apply BCG matrix framework from Skills
5. Stream response with citations → "According to [Oracle vs Salesforce case, p.4]..."
6. Log to evaluation pipeline
```

**Safety (OpenAI "dumb guardrails" approach):**
- Source attribution required on every factual claim
- Confidence threshold: retrieval score < 0.7 → "I'm not certain — here's what I found..."
- Hard scope boundary: business case analysis only — graceful decline for off-topic
- Simple, strict rules — not clever ones that fail at edges

**Fine-tuning / Preference Optimization:**
- Not yet — insufficient interaction data
- DPO becomes viable at ~10K rated interactions
- Current approach: few-shot examples in system prompt + Skills templates achieves comparable quality at zero fine-tuning cost

---

## Slides 9-10: Cost & Revenue [2.5 pts]

### Assumptions: 100 Active Users
- Consulting prep platform: 3 sessions/week, 15 queries/session
- 100 × 3 × 15 × 4.3 weeks = **~19,350 queries/month**

---

### Cost Model

**Tiered model routing (60/40 split):**
- 60% simple factual → **Claude Haiku** ($1 input / $5 output per MTok)
- 40% complex synthesis → **Claude Sonnet** ($3 input / $15 output per MTok)
- Average per query: 2,500 input + 600 output tokens

| Segment | Queries | Input | Output | Cost |
|---|---|---|---|---|
| Haiku (60%) | 11,610 | 29.0 MTok | 7.0 MTok | $29 + $35 = **$64** |
| Sonnet (40%) | 7,740 | 19.4 MTok | 4.6 MTok | $58 + $69 = **$127** |
| Prompt cache savings | — | — | — | **-$20** |
| **LLM total** | | | | **$171** |

**Full monthly cost breakdown:**

| Item | Monthly Cost |
|---|---|
| LLM API (tiered Haiku/Sonnet + prompt caching) | $171 |
| Pinecone Starter (5 PDF cases, ~50K chunks) | $70 |
| Text embedding API (case updates + new cases) | $5 |
| Bing News Search API | $25 |
| Supabase (user memory storage) | $0 (free tier) |
| Vercel Pro (hosting + serverless) | $20 |
| **Total** | **~$291/month** |

**Per-user cost: ~$2.91/month**

---

### Revenue Model

| Metric | Value |
|---|---|
| Price | $20/month (vs. $200+/hr coach, $60-90/month RocketBlocks) |
| Revenue at 100 users | $2,000/month |
| Gross profit | $1,709/month (**85% gross margin**) |
| Break-even | **15 paying users** |

---

### Cost Tradeoffs

**1. Tiered routing risk**
Routing 60% of queries to Haiku cuts LLM cost ~40%, but Haiku may under-perform on complex synthesis.
*Mitigation:* Quality gate — if Haiku confidence score < threshold, auto-escalate to Sonnet.

**2. Prompt caching vs. personalization**
Caching works when the system prompt is identical across requests. Adding per-user memory fragments the cache, eroding savings.
*Tradeoff:* Keep system prompt fixed; inject user memory as a separate (non-cached) prefix. Saves ~$20/month while preserving personalization.

**3. Scale cliff at 10,000 users**
Pinecone Starter handles ~100K vectors. At 10,000 users, Pinecone Standard costs $500+/month.
*Mitigation:* Self-host Weaviate on a $50/month VPS — same capability, 10x cheaper at scale.

**4. Seasonal demand risk**
Consulting recruiting peaks Sep-Nov and Jan-Mar. Monthly active users vary 3-4x throughout the year.
*Mitigation:* Offer per-session pricing ($1.50/session) alongside subscription — smooths revenue without locking students into off-season subscriptions.

**5. News API bottleneck**
Bing free tier caps at 3 API calls/second. At high query volume with real-time news enabled, this creates latency.
*Mitigation:* Cache news results by topic for 4 hours; rate-limit news tool to complex synthesis queries only.

---

## Speaker Notes: Key Talking Points

1. **Lead with the live product** — show oracle-salesforce-dashboard.vercel.app. This is deployed, not hypothetical. The gap between it and the agentic vision is the story.

2. **The ToDo as narrative bridge** — the repo literally has a file that says "Task 1: Let's make this a RAG system." The agentic system is the natural evolution.

3. **The OpenAI architecture parallel** — explicitly name the 6-layer mapping. Shows you read the article and applied it, not just summarized it.

4. **Evaluation is the differentiator** — most presentations will say "pilot group" or "user testing." Six-layer automated eval with CI/CD gating is operationally specific and impressive.

5. **The monetization reframe** — helping students study is the free tier. Replacing $200/hr coaches with a $20/month subscription is the revenue thesis. That's a real business.

6. **Context curation > context volume** — use this line. It's the intellectual core of the RAG argument and it comes from the OpenAI article directly.
