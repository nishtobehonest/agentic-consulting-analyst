# Evaluation Framework — Agentic Consulting Analyst

## Core Principle (from OpenAI's in-house data agent)

> "The only way to scale capability without breaking trust is through systematic evaluation."
> — OpenAI, "Inside our in-house data agent"

OpenAI's approach: curated Q&A pairs with "golden answers," compared programmatically using semantic equivalence (not naive string matching). The agent can produce correct answers in many forms — evaluation must recognize correctness, not just exact wording.

I adapt and extend this to case analysis.

---

## Why Not a Pilot Group?

Pilot groups don't scale for 3 reasons:
1. **Slow feedback loop** — recruiting users, running sessions, collecting feedback takes weeks
2. **Inconsistent signals** — users give varied, subjective feedback that's hard to aggregate
3. **Cannot gate deploys** — you can't run a pilot group on every code push

My 6-layer automated framework runs in <5 minutes on every git push, produces quantitative metrics, and blocks bad deploys programmatically.

---

## 6-Layer Evaluation Framework

---

### Layer 1: Golden Answer Benchmarks

**What it tests:** Factual accuracy and analytical correctness

**How it works:**
1. Curate 150 Q&A pairs with verified answers from case PDFs and `caseData.js`
2. Two categories:

**Factual questions** (exact verifiable answers):
```
Q: "What was Salesforce's total revenue in 2020?"
A: "$17.1 billion"
Source: caseData.js, financialData.salesforce[year=2020].revenue

Q: "In what year did Oracle acquire NetSuite?"
A: "2016"
Source: oracle-vs-salesforce.pdf, acquisitions section

Q: "What was Oracle's cloud revenue growth rate in 2023?"
A: "Retrieved from live financial API, cross-checked against case"
```

**Analytical questions** (framework-grounded, scored by semantic similarity):
```
Q: "What is Oracle's primary competitive moat in the enterprise software market?"
Golden answer (authored by course professor or analyst):
  "Oracle's primary moat is high switching costs from deeply embedded ERP and
   database integrations, combined with long-term customer contracts that create
   revenue predictability. This is reinforced by the Oracle ecosystem lock-in..."

Scoring: cosine similarity of answer embedding vs. golden answer embedding
Threshold: similarity > 0.75 → PASS
```

**Infrastructure:**
- Test harness: Python script (pytest-style) triggered via CI/CD on every push
- Embeddings: OpenAI text-embedding-3-small for semantic similarity scoring
- Results: JSON report with per-question pass/fail + similarity scores
- Dashboard: Score trend over time — catch regressions before deploy

**Target metrics:**
- Factual accuracy: >95% exact match
- Analytical similarity: >80% questions above 0.75 cosine threshold

---

### Layer 2: RAG Retrieval Quality

**What it tests:** Does the retrieval system surface the right evidence?

**How it works:**
1. Build a retrieval test set: 50 questions with known source passage locations in PDFs
2. Example:
```
Q: "What was Salesforce's strategy for moving into enterprise accounts?"
Known source: oracle-vs-salesforce.pdf, page 8, paragraph 3
             "Salesforce's land-and-expand strategy..."
```
3. Run retrieval; check if the known passage appears in top-k results

**Metrics:**
- **Recall@5:** Does the correct passage appear in the top 5 retrieved chunks? (Target: >85%)
- **Precision@1:** Is the correct passage the top-ranked result? (Target: >65%)
- **MRR (Mean Reciprocal Rank):** Average of 1/rank of first correct result (Target: >0.7)

**Why this matters:** If retrieval fails, the agent will hallucinate or use irrelevant context even if the LLM is perfect. This layer catches embedding quality issues, chunking strategy failures, and query-passage mismatch.

**Zero humans required:** Pure vector similarity comparison, fully automated.

---

### Layer 3: LLM-as-Judge

**What it tests:** Response quality on dimensions humans care about

**How it works:**
- 5% random sample of live production queries (after rollout)
- Or: 20 held-out "gold evaluation queries" run on every deploy
- Claude Opus evaluates each response on 4 dimensions (1-5 scale):

```
Evaluation prompt to Opus:
"""
You are evaluating a response from an AI consulting analyst.

CASE DATA (ground truth):
{relevant_case_content}

USER QUESTION:
{user_question}

AGENT RESPONSE:
{agent_response}

Rate this response on a 1-5 scale for each dimension:

1. FACTUAL ACCURACY: Are all claims consistent with the case data?
   (1=contradicts case data, 5=perfectly accurate with citations)

2. CITATION QUALITY: Does the response cite specific sources?
   (1=no citations, 5=every factual claim has page/section reference)

3. ANALYTICAL DEPTH: Does the response apply appropriate frameworks?
   (1=surface level, 5=deep framework application with nuanced insight)

4. PEDAGOGICAL CLARITY: Is this helpful for a student learning?
   (1=confusing, 5=clear explanation that builds mental models)

Return JSON: {"factual": N, "citation": N, "analytical": N, "pedagogical": N, "notes": "..."}
"""
```

**Infrastructure:**
- Async evaluation job (doesn't block user response)
- Results stored in Supabase `evaluations` table
- Weekly summary report: moving average of all 4 dimensions
- Alert if any dimension drops below 3.5 average

**Why Opus as judge:** Using a stronger model than the agent (Sonnet/Haiku) as evaluator mirrors human judgment better. Directly analogous to OpenAI's "compare generated SQL output to golden SQL output."

---

### Layer 4: Consistency / Stability Testing

**What it tests:** Does the agent give stable, non-hallucinatory answers?

**How it works:**
- Run 30 representative questions × 10 times each
- Compare answers using embedding cosine similarity
- Flag questions with high variance

```python
# Pseudocode
for question in test_questions:
    answers = [agent.query(question) for _ in range(10)]
    embeddings = [embed(a) for a in answers]
    variance = mean_pairwise_distance(embeddings)
    if variance > THRESHOLD:
        flag_for_inspection(question, answers, variance)
```

**Targets:**
- Factual questions: <15% variance (answers should be nearly identical)
- Analytical questions: <30% variance (some phrasing variation is fine, substance should be stable)

**Why this matters:** High variance is a proxy for "the model doesn't know and is guessing differently each time." Low-variance wrong answers are easier to fix than high-variance answers.

---

### Layer 5: Hallucination Trap Detection

**What it tests:** Does the agent accept false premises or correct them?

**How it works:**
20 trap questions with deliberately false premises injected into the test set:

```
TRAP QUESTIONS (examples):

"Given that Oracle's revenue declined 30% in 2022, what caused this downturn?"
→ Expected: "Actually, Oracle's revenue grew X% in 2022 according to the case data..."
→ Fail case: "Oracle's revenue declined because of cloud competition..."

"Salesforce acquired Oracle in 2019 — how did this affect their go-to-market strategy?"
→ Expected: "This is incorrect. Salesforce did not acquire Oracle..."
→ Fail case: "After the acquisition, Salesforce integrated Oracle's..."

"The case study shows Oracle has no cloud products as of 2024..."
→ Expected: "That's not accurate — the case data shows Oracle Cloud Infrastructure..."
→ Fail case: "That's right, Oracle has struggled to enter the cloud market..."
```

**Scoring:**
- PASS: Agent explicitly corrects the false premise before answering
- FAIL: Agent accepts the premise and builds a response on false information
- Target: >95% correction rate on 20 traps

**Why this matters:** Hallucination traps are the most dangerous failure mode for a case interview prep tool. If a student gets confidently wrong analysis, they'll repeat it in their McKinsey interview.

---

### Layer 6: Scope Boundary Testing

**What it tests:** Does the agent stay on-topic and degrade gracefully?

**Key OpenAI insight:**
> "Reliability degrades quickly when scope expands faster than governance."
They deliberately avoided giving broad autonomy. My agent should have strict, "dumb guardrails."

**Test categories:**

**Off-topic queries (should decline gracefully):**
```
"What's the best restaurant in Manhattan?" → Expected: "I'm focused on business case analysis..."
"Help me write my cover letter" → Expected: "That's outside my scope..."
"What's the weather like?" → Expected: brief, polite decline
```

**Ambiguous queries (should ask for clarification, not hallucinate):**
```
"Compare their strategies" (no case context) → "Which cases are you comparing?"
"What happened next?" (no prior context) → "Could you clarify what you're referring to?"
```

**Out-of-knowledge queries (should say "I don't know"):**
```
"What did Oracle's CEO say in last week's earnings call?" → "I don't have access to that..."
"What's the acquisition price Oracle paid for a company not in the case?" → "I don't have that specific data..."
```

**Targets:**
- Off-topic decline rate: >98%
- Ambiguous query clarification rate: >85% (rather than guessing)
- Out-of-knowledge acknowledgment rate: >90% (rather than hallucinating)

---

## Evaluation Pipeline Architecture

```
git push
    │
    ▼
CI/CD Pipeline (GitHub Actions / Vercel Deploy Hook)
    │
    ├── Layer 1: Golden Answer Benchmarks     (~2 min)
    │   └── Run 150 Q&A pairs → JSON report
    │
    ├── Layer 2: RAG Retrieval Quality        (~1 min)
    │   └── 50 retrieval queries → recall/precision report
    │
    ├── Layer 4: Consistency Testing          (~3 min)
    │   └── 30 questions × 10 runs → variance report
    │
    ├── Layer 5: Hallucination Traps          (~30 sec)
    │   └── 20 trap questions → correction rate
    │
    └── Layer 6: Scope Boundary Tests         (~1 min)
        └── 30 boundary queries → decline/clarify rate

Total automated eval time: ~8 minutes per deploy

GATE:
  All metrics above threshold → DEPLOY to production
  Any metric below threshold → BLOCK deploy + notify
```

**Layer 3 (LLM-as-Judge):** Runs async on live traffic post-deploy — too expensive to run on every push. Results inform next iteration.

---

## Metrics Dashboard

Track these over time (plot weekly):

| Metric | Target | Alert Below |
|---|---|---|
| Factual accuracy (Layer 1) | >95% | <90% |
| Analytical similarity (Layer 1) | >80% at 0.75 cosine | <70% |
| RAG Recall@5 (Layer 2) | >85% | <75% |
| RAG Precision@1 (Layer 2) | >65% | <55% |
| LLM-Judge avg score (Layer 3) | >4.0 / 5.0 | <3.5 |
| Answer variance (Layer 4) | <15% factual | >25% |
| Hallucination trap pass (Layer 5) | >95% | <90% |
| Scope decline rate (Layer 6) | >98% | <95% |

---

## Connecting to Course Concepts

| Concept | How it applies here |
|---|---|
| **RAG** | Layer 2 directly measures RAG quality — retrieval is the foundation everything else depends on |
| **Memories** | Layer 4 tests stability, which is also how you validate that memory injection isn't corrupting answers |
| **Fine-tuning** | Evaluation data (especially Layer 3 LLM-as-judge scores) becomes preference pairs for future DPO fine-tuning |
| **Preference Optimization** | Layer 3 scores are the ground truth for preference data — [good response, bad response] pairs for RLHF/DPO |
| **Tools** | Layer 5 and Layer 6 test tool invocation boundaries — when the agent should use tools vs. decline |
| **Skills** | Layer 1 analytical questions test whether skills (framework templates) are applied correctly |
