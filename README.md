# 🎤  Multi-Agent AI Event Planning System

> A stateful, production-grade multi-agent AI system that turns fragmented event data into reliable, explainable planning decisions.

---

## 🧠 The Problem

Event planning at Live Nation isn't just scheduling — it's optimizing across multiple dimensions simultaneously:

- Venue constraints (capacity, availability, layout)
- Food & beverage sales performance
- Upsell and merchandise conversion
- Fan satisfaction and feedback
- Revenue per event and per artist

All of this data existed — but it was **fragmented** across SQL systems, unstructured notes, Slack threads, and spreadsheets. Planning was manual, slow, and brittle when inputs changed.

A single LLM call couldn't handle this. The problem required different types of reasoning: retrieval, planning, validation, and explanation. So we built a proper system.

---

## 🏗️ Architecture

```
User Input (Slack)
        │
        ▼
┌───────────────────────────────────────────────┐
│             LangGraph Orchestrator             │
│                                               │
│  ┌──────────────────┐                         │
│  │   Intent Agent   │ ◄── Understands goal,   │
│  │                  │     scope, entities      │
│  └────────┬─────────┘                         │
│           │  Out of scope? ──► Clarification  │
│           │                       Agent ──┐   │
│  ┌────────▼─────────┐                    │   │
│  │  Databricks Genie│ ◄── Generates SQL  │   │
│  │   (SQL Agent)    │     + executes it  │   │
│  └────────┬─────────┘                    │   │
│           │                              │   │
│  ┌────────▼─────────┐  (parallel)        │   │
│  │  Reasoning Agents│                    │   │
│  │  ┌─────────────┐ │                    │   │
│  │  │Venue Analysis│ │                    │   │
│  │  ├─────────────┤ │                    │   │
│  │  │Food & Stock  │ │                    │   │
│  │  ├─────────────┤ │                    │   │
│  │  │Upsell/Merch  │ │                    │   │
│  │  ├─────────────┤ │                    │   │
│  │  │Fan Sentiment │ │                    │   │
│  │  └─────────────┘ │                    │   │
│  └────────┬─────────┘                    │   │
│           │                              │   │
│  ┌────────▼─────────┐                    │   │
│  │Summarization Agent│                   │   │
│  └────────┬─────────┘                    │   │
│           │                              │   │
│  ┌────────▼─────────┐                    │   │
│  │ Validation Agent │ ◄── Constraint     │   │
│  └────────┬─────────┘     checking       │   │
│           │  Fail? ──► loop back         │   │
│  ┌────────▼─────────┐                    │   │
│  │ Explanation Agent│ ◄── Final output   │   │
│  └──────────────────┘         ▲          │   │
│                                └──────────┘   │
└───────────────────────────────────────────────┘
        │
        ▼
  Planner (via Slack)
```

**Key design principle: The graph controls execution. Not the LLM.**

---

## 🤖 Agent Pipeline

### 1 — Intent Agent
- First agent in every workflow — understands what the planner is actually asking
- Extracts: goal, relevant entities (venue, artist, date range), domain (venue / food / upsell / fan)
- Detects out-of-scope or ambiguous questions → routes to Clarification Agent before any data is fetched
- Scopes the downstream agents — only the relevant reasoning agents are activated per request

### 2 — Databricks Genie (SQL Agent)
- Receives structured intent from the Intent Agent
- Generates a SQL query using Databricks Genie's natural language → SQL capability
- Validates the query before execution (read-only, row-limited, aggregation-enforced — see SQL Guardrails)
- Executes against Databricks tables and returns structured results
- Retries with corrected SQL if execution fails, using exponential backoff

### 3 — Reasoning Agents *(run in parallel)*

| Agent | What it does |
|---|---|
| **Venue Analysis** | Capacity utilisation, layout fit, historical performance at venue, availability constraints |
| **Food & Stock Planning** | F&B sales patterns, stock recommendations by event type/crowd size, waste reduction |
| **Upsell & Merchandise** | Conversion rates by product/artist, upsell timing, bundle opportunities |
| **Fan Satisfaction** | Sentiment from feedback data, common complaints, satisfaction trends by venue or artist |

- Each agent receives only the data slice it needs (prompt compression)
- Agents run in parallel — results merged before summarization
- Each produces a structured output: findings + confidence score + data gaps

### 4 — Summarization Agent
- Merges outputs from all active reasoning agents into a single coherent context
- Removes redundancy, resolves conflicts between agent outputs
- Compresses the merged result before passing to validation — keeps token usage lean
- Flags any low-confidence findings for the validation agent to scrutinize

### 5 — Validation Agent
- Checks the summarized plan against hard constraints (venue capacity, budget limits, date feasibility)
- Rejects invalid plans with structured feedback → loops back to reasoning agents
- Confirms SQL results were actually used (not fabricated by the LLM)
- Escalates to human review if validation fails after max retries

### 6 — Explanation Agent
- Produces final output in plain language — what the recommendation is and *why*
- Cites the data sources and agent findings behind each decision
- Attaches confidence level and any caveats
- Sends result back to the planner via Slack

### 7 — Clarification Agent *(on-demand)*
- Triggers when: intent is ambiguous, data is insufficient, question is out of scope, or confidence is below threshold
- Sends a **targeted question** to the planner via Slack instead of guessing
- Workflow pauses and resumes from the Intent Agent once the planner responds
- Prevents low-quality or hallucinated answers from reaching planners silently

---

## 📦 Data & Retrieval Layer

The system works over two data types:

**Structured**
- SQL tables: revenue, product sales, venue data, artist history

**Unstructured**
- Fan feedback, event notes, artist-specific insights

**Hybrid retrieval pipeline:**
1. SQL queries for exact metrics
2. Vector search for semantic context (fan feedback, similar events)
3. Re-ranking to surface the most relevant results

All recommendations are grounded in real data — no hallucinated strategies.

---

## 🗂️ State Management

Each request maintains a **shared state object** containing:

```python
{
  "user_input": "...",
  "constraints": {...},
  "retrieved_data": {...},
  "intermediate_plan": {...},
  "validation_results": {...},
  "decision_summary": "..."
}
```

**Checkpointing** is added after every step. If something fails mid-workflow, the system resumes from that step — not from scratch. This:

- Reduces cost (no re-running expensive retrieval)
- Enables partial retries
- Makes debugging significantly easier

**Context management:** Previous steps are summarized before being passed forward. The system always has just enough memory to reason correctly — without bloating the context window.

---

## 🔒 Reliability & Error Handling

Production failures are expected, not edge cases. The system is designed around this:

- Every agent has **strict input/output schemas**
- **Timeouts and retry policies** on all agents
- **Deterministic tools** for critical operations (revenue queries use SQL, not LLM inference)
- **Validation layers** catch incorrect outputs before they propagate

If the planning agent produces something infeasible → validation rejects it with structured feedback → planning retries with that context.

If **confidence is low or data is missing** → the system escalates to human review instead of guessing.

---

## ⚡ Deployment & Concurrency

**Deployed as a Slack application** — planners already lived there, so zero behavior change required.

### Across requests

```
Slack Event → Queue → Async Worker → Workflow Execution
```

Prevents timeouts. Allows horizontal scaling.

### Within a request

- Independent tasks (SQL queries, vector retrieval) run **in parallel**
- Each Slack thread holds a **lock** — only one workflow updates state at a time
- **Idempotency keys** prevent Slack's retry mechanism from triggering duplicate workflow runs

---

## 📈 Scaling & Evaluation

**Scaling strategy: decomposition, not bigger prompts.**

When complexity increased, we added new agents and extended workflows — we did not increase prompt size.

**Evaluation pipeline (Databricks):**
- Replay historical scenarios against the system
- Measure recommendation accuracy, constraint violations, output consistency
- Monitor production metrics: latency, retry rates, validation failure rates

---

## 📊 Results

| Metric | Impact |
|---|---|
| Planning speed | ~30% improvement |
| Decision consistency | Standardized across planners and events |
| Explainability | Every recommendation includes traceable reasoning |
| Adaptability | Inputs (venue swap, artist cancel) handled without restarting workflows |

---

## 🧪 Validation & Testing

- Built a **sample dataset** covering venues, artists, revenue, and fan feedback to validate each agent independently
- Each agent tested in isolation — inputs, outputs, and edge cases verified before integration
- End-to-end workflow replayed against historical scenarios to confirm consistency
- **MLflow** used for experiment tracking: logged agent inputs/outputs, latency, token usage, retry counts, and validation pass/fail rates per run
- Production metrics monitored via MLflow dashboards — latency, failure rates, and recommendation accuracy tracked over time

---

## 🗄️ State Management — PostgreSQL

- Workflow state persisted in **PostgreSQL** (not in-memory) — survives restarts, supports resume from any checkpoint
- Each workflow run stored with a unique ID; agents read/write state to the same row
- Enables full audit trail — every intermediate state, plan, and validation result is queryable
- Horizontal scaling works cleanly — any worker can pick up any workflow by querying state from Postgres

---

## 🗜️ Prompt Compression

- Only **relevant, required information** is passed to each agent — not full history
- Previous steps are **summarized**, not appended raw — keeps context windows lean
- Each agent receives only what it needs: retrieval agent gets the query, planning agent gets retrieved data, validation agent gets the plan + constraints
- Token usage tracked per agent via MLflow — compression reduced costs significantly without accuracy loss

---

## 🛡️ SQL Safety & Query Guardrails

- All SQL queries are **read-only** — `INSERT`, `UPDATE`, `DELETE` operations are blocked at the query execution layer
- Queries enforced with **row limits** (`LIMIT N`) — no unbounded table scans
- Only **aggregations** allowed on large tables (e.g., `SUM`, `AVG`, `COUNT`) — raw row dumps restricted
- Query validator checks generated SQL before execution — rejects anything outside the allowed pattern
- Prevents both accidental data mutation and runaway queries from hitting production tables

---

## 🔁 Retry Mechanism — Exponential Backoff

- Every agent wrapped with **exponential backoff** retry logic
- On failure: waits `2^n` seconds before retrying (e.g., 1s → 2s → 4s → 8s...)
- Max retry attempts capped — after exhausting retries, escalates to human review
- Transient failures (timeouts, rate limits) handled silently; persistent failures surfaced with full context
- Retry counts and backoff intervals logged in MLflow per agent per run

---

## ❓ Clarification Agent

- Dedicated agent that triggers when:
  - Retrieved data is **insufficient** to answer the question
  - The question is **out of scope** (not related to event planning data)
  - Confidence in the plan is below threshold after retries
- Instead of guessing or hallucinating, the system **asks the planner a targeted clarifying question** via Slack
- Clarification responses are fed back into the workflow and re-run from the retrieval step
- Prevents low-quality recommendations from reaching planners silently

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | LangGraph |
| LLM | Claude / GPT-4 |
| Structured data | SQL (Databricks) |
| Semantic search | Vector DB + re-ranking |
| State persistence | PostgreSQL |
| Experiment tracking | MLflow |
| Deployment | Slack App + async workers |
| Evaluation | Databricks replay pipelines + sample datasets |

---

## 💡 Key Lessons

Building AI systems in production is not about the model.

It's about:

1. **State management** — knowing what happened, resuming where you left off
2. **Orchestration** — deterministic control flow, not LLM-driven chaos
3. **Reliability** — designing for failure as a normal case, not an exception
4. **Knowing when not to answer** — escalating to humans when confidence is low

The model is ~20% of the work. The infrastructure around it is the other 80%.

---

## 🚀 One-liner

> A stateful, multi-agent system that turned fragmented event data into reliable, explainable planning decisions — production-ready with proper orchestration, concurrency handling, and failure recovery.
