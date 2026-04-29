# dbFlow — Conversational Flow Intelligence Platform
## Implementation Blueprint for Deutsche Bank Innovation World Cup

> **Author:** Subhash | VP Technology, Deutsche Bank India  
> **Category:** Data-Driven Opportunities  
> **Comparable:** Goldman Sachs Marquee  
> **Target:** Hedge Funds · Sovereign Wealth Funds · Corporate Treasuries

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The Core Idea](#2-the-core-idea)
3. [Why Only DB Can Build This](#3-why-only-db-can-build-this)
4. [Technical Architecture](#4-technical-architecture)
5. [18-Month Implementation Roadmap](#5-18-month-implementation-roadmap)
6. [Team and Skills Required](#6-team-and-skills-required)
7. [Pricing and Revenue Model](#7-pricing-and-revenue-model)
8. [Hard Questions — Answered](#8-hard-questions--answered)
9. [My Role and Positioning](#9-my-role-and-positioning)
10. [First Step — Week 1 Action Plan](#10-first-step--week-1-action-plan)

---

## 1. Executive Summary

dbFlow is an AI-native, conversational data intelligence platform that productizes Deutsche Bank's institutional flow data — FX, trade finance, correspondent banking, and prime brokerage — for external institutional clients.

**The pitch in one sentence:** We train an LLM on DB's real-money flow data and let hedge funds, sovereign wealth funds, and corporate treasuries query it conversationally — the same way a senior DB analyst would brief them, but available 24/7 via API.

| Metric | Target |
|--------|--------|
| Year 1 ARR (conservative) | €3.5M |
| Year 2 ARR | €20M |
| Year 3 ARR trajectory | €100–200M |
| Gross margin | 70–85% |
| Time to first demo | 8 weeks |
| Time to first paying client | 12 months |
| Peak team size (18 months) | 7–8 people |

---

## 2. The Core Idea

### What it does

A hedge fund quant opens a Jupyter notebook and types:

```python
import dbflow
result = dbflow.query("What is the real-money positioning in EM FX this week?")
print(result.narration)   # LLM-generated insight paragraph
print(result.signals)     # Underlying flow signal rows (cited)
print(result.sources)     # Research context used
```

They get back a structured answer grounded in DB's actual institutional flow data — not a Bloomberg price feed, not a public report, but real-money signals from the bank that processes trillions in FX daily.

### Three demo scenarios for the Innovation World Cup

1. **"What is the FX positioning in EM currencies this week?"**  
   → LLM synthesizes DB's Autobahn FX flow aggregates + macro research context → structured answer with directional bias, notional band, sector breakdown

2. **"Show me trade finance flow trends in Asia Pacific"**  
   → Pulls from the Corporate Bank's trade finance Iceberg tables → regional heat map of LC issuance, supply chain finance volumes, counterparty sector trends

3. **"What are hedge funds doing in European credit?"**  
   → Prime brokerage positioning data → net exposure changes, crowding signals, sector rotation narrative

### What makes it different from Bloomberg

Bloomberg sells the **same public data to everyone** — it is a commodity reference, not an alpha source. dbFlow sells **signal that is inherently exclusive**. When a hedge fund builds a model on DB's FX flow data, they are seeing where real institutional money moved — not a reconstructed estimate, not a public price feed. The correct pricing analogy is prime brokerage research, not terminal access.

---

## 3. Why Only DB Can Build This

### The data moat

| Data Asset | Source System | Why It Matters |
|------------|--------------|----------------|
| FX flow data | Autobahn platform | DB processes trillions in FX daily — directional, notional-banded, real-money |
| Trade finance flows | Corporate Bank (letters of credit, LCs) | Cross-border supply chain signals no public dataset captures |
| Prime brokerage positioning | Hedge fund client book | Net/gross leverage, sector rotation, crowding signals |
| Correspondent banking flows | Cross-border payment rails | Early macro indicator for EM currency pressure |

No fintech startup can build this. Goldman Marquee took 10+ years. DB already has the data and the GKE/GCP infrastructure. What's missing is the LLM intelligence layer and the client-facing API — which is exactly what dbFlow delivers.

### The infrastructure moat (already built)

The modern data platform already running in production:
- **Apache Iceberg** — feature store with time-travel for point-in-time correct training data
- **Trino** — federated query across all source systems in one SQL statement
- **Nessie** — Git-like catalog versioning; every model training run checkpoints the exact data version
- **Airflow on GKE** — orchestration, already running, KubernetesPodOperator for isolation
- **Ollama + Gemma** — air-gap approved LLM infrastructure (Central Architecture Board approved)

**70% of the infrastructure already exists.** This is not a greenfield project — it is a product layer on top of proven foundations.

---

## 4. Technical Architecture

### Layer 1 — Proprietary data layer (the moat)

```
FX flows (Autobahn) → Trade finance (LCs) → Prime brokerage → Correspondent banking
         ↓                    ↓                    ↓                   ↓
                    [Aggregated, non-attributable flow signals]
                    [No raw client data. No position-level PII.]
```

**Critical design decision:** All signals are aggregated at a level that passes DB data classification. Same level of aggregation as DB's own published quarterly flow reports — just queryable in real time. This is the answer to every compliance question in the first 60 days.

### Layer 2 — Feature store (Iceberg schema design)

```sql
-- Core flow signal table
CREATE TABLE dbflow.signals.fx_flows (
    signal_date       DATE,
    asset_class       VARCHAR,        -- 'FX', 'RATES', 'CREDIT'
    region            VARCHAR,        -- 'EMEA', 'APAC', ' Americas'
    direction         VARCHAR,        -- 'BUY', 'SELL', 'NET'
    counterparty_sector VARCHAR,      -- 'HEDGE_FUND', 'REAL_MONEY', 'CORPORATE'
    notional_band     VARCHAR,        -- 'SMALL', 'MEDIUM', 'LARGE' (never raw notional)
    flow_score        DOUBLE,         -- normalized -1.0 to +1.0
    confidence        DOUBLE,         -- model confidence on aggregation
    updated_at        TIMESTAMP
)
PARTITIONED BY (signal_date, asset_class, region);
```

**Why notional_band instead of raw notional:** Prevents reverse-engineering of individual client positions. Passes data classification. Still conveys directional strength.

**Airflow DAG (daily):**
```
Source Trino query → Aggregation logic → PII check → Classification validation → Iceberg write → Nessie commit → Quality check → Alert on anomaly
```

### Layer 3 — LLM intelligence (the AI core)

#### Component decisions

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Base model | Gemma 2 9B via Ollama | Already air-gap approved. No re-approval needed. |
| Fine-tuning | LoRA via Hugging Face PEFT | Adds DB financial vocabulary in a 50MB adapter. Full fine-tune unnecessary and expensive. |
| Retrieval | pgvector + LangChain RAG | Grounds every answer in cited source rows. Hallucination control. |
| Guardrails | Custom pre/post-generation scanner | Non-negotiable for external product. See below. |

#### RAG pipeline (query time)

```
User query (NL)
      ↓
Query classifier — is this asking for client-identifiable information?
      ↓ (if safe)
Embedding → pgvector similarity search → top-K research chunks
      ↓
Trino query → relevant flow signal rows for the date range and asset class
      ↓
LangChain chain: [system prompt] + [retrieved research] + [flow signals] + [user query]
      ↓
Gemma 2 9B generation (streaming)
      ↓
Post-generation scanner — PII check, client name check, position size check
      ↓
Structured response: {narration, signals[], sources[], confidence}
```

#### Guardrails layer (non-negotiable)

**Pre-generation (query classifier):**
- Blocks queries asking about specific client names, account IDs, or individual position sizes
- Flags queries that could be used to reverse-engineer proprietary positions
- Returns an error with explanation, not a hallucinated safe answer

**Post-generation (output scanner):**
- Scans for PII patterns (names, account numbers, exact notional figures)
- Verifies every factual claim is traceable to a cited source row
- Confidence scoring: if model confidence < threshold, adds uncertainty disclaimer

**Legal framing:** dbFlow is an intelligence tool, not investment advice. Same legal framing Bloomberg Terminal uses. Every response carries a standard disclaimer. The client makes the trade — dbFlow informs the research process.

### Layer 4 — API and delivery layer

#### FastAPI + GKE

```python
# API contract — what clients actually call
POST /v1/query
{
  "query": "What is the real-money FX positioning in EM this week?",
  "asset_classes": ["FX", "RATES"],
  "regions": ["EMEA", "APAC"],
  "date_range": {"from": "2025-04-01", "to": "2025-04-29"},
  "stream": true
}

# Response structure
{
  "narration": "Real-money flows in EM FX showed net selling...",
  "signals": [{"date": "...", "region": "EMEA", "flow_score": -0.42, ...}],
  "sources": [{"type": "research", "title": "...", "date": "..."}, ...],
  "confidence": 0.87,
  "disclaimer": "For informational purposes only. Not investment advice."
}
```

#### Python SDK (what quant teams actually use)

```python
pip install dbflow-sdk

from dbflow import Client
client = Client(api_key="your_key")

# Single query
result = client.query("EM FX positioning this week")

# Streaming
for token in client.stream("What are hedge funds doing in European credit?"):
    print(token, end="", flush=True)

# Scheduled signal alerts
client.subscribe(
    signal="FX_EM_NET_FLOW",
    threshold=-0.5,    # alert when EM FX flow score drops below -0.5
    webhook="https://your-system.com/alerts"
)
```

#### Auth model

| Tier | Auth mechanism | Rate limit |
|------|---------------|-----------|
| Tier 1 | Per-user API key | 100 queries/day |
| Tier 2 | Firm OAuth2 + per-user keys | Unlimited, SLA-backed |
| Tier 3 | Dedicated namespace, dedicated endpoint | Custom SLA |

**Billing entity = firm (OAuth2). Usage identity = individual analyst (API key).** Required for multi-user enterprise contracts.

---

## 5. 18-Month Implementation Roadmap

### Phase 0 — Foundation (Weeks 1–8) · Solo

> **The answer when asked "what is your first step?"**

| Week | Action | Output |
|------|--------|--------|
| 1–2 | Data inventory and classification audit | One-pager: approved datasets, schemas, classification levels |
| 3–4 | Feature store design and Airflow DAG | Iceberg flow signal tables populated daily |
| 5–6 | RAG layer over internal research corpus | LangChain chain returning grounded answers |
| 7–8 | Live demo interface (React) | Working demo: 3 canned scenarios, streaming responses |

**Why data inventory first, not coding:** You do not touch application code until you know exactly which data is approved for use. This is not bureaucracy — it is the credibility signal that separates engineers from product leaders. If you demo with non-approved data and compliance finds out, the project is dead in week 9.

**Phase 0 deliverable = Innovation World Cup demo.** A working system, not a slide deck.

---

### Phase 1 — PoC to Internal Pilot (Months 3–6) · You + 2 people

**New hires:** 1 FIC quant analyst (domain expert), 1 ML engineer

| Month | Action | Output |
|-------|--------|--------|
| 3 | First internal pilot — FIC desk | Daily usage, feedback loop, accuracy baseline |
| 4 | LoRA fine-tuning pipeline | DB-vocabulary-tuned Gemma adapter on GKE |
| 5 | Multi-asset expansion | Trade finance + correspondent banking modules live |
| 6 | Architecture board review | Formal sign-off to proceed to external pilot |

**Success metrics for Month 6 review:**
- Query volume: >500 queries/week from pilot traders
- Answer accuracy rate: >80% vs senior analyst benchmark
- Time-to-insight: <30 seconds vs 2-4 hours manual research
- NPS from pilot users: >40

---

### Phase 2 — External Alpha (Months 7–12) · You + 5 people

**New hires:** 1 compliance specialist, 1 backend engineer

| Month | Action | Output |
|-------|--------|--------|
| 7–8 | MiFID II compliance layer + data governance | Every signal row traceable. External sharing approved. |
| 9 | External API gateway + Python SDK | Rate-limited, multi-tenant, OAuth2-secured |
| 10–11 | Alpha client onboarding (5 clients, free 90-day) | Weekly feedback sessions, usage analytics |
| 12 | First paid contracts | 2–3 clients × $50k–$200k/yr = first ARR |

**The Month 12 moment is the most important milestone in the project.** Revenue is proof. Everything before this is cost.

---

### Phase 3 — Productization and Scale (Months 13–18) · Full team

**New hires:** 1 additional ML engineer, 1 frontend/UX, 1 client success manager

| Month | Action | Output |
|-------|--------|--------|
| 13–14 | Self-service client portal | API key management, billing, module selection — no human onboarding needed |
| 15 | Bespoke Tier 3 delivery | Custom model fine-tuned per SWF client. $1–5M/yr contracts. |
| 16–18 | Monetization flywheel | Usage data → signal improvement → better model → more clients |

---

## 6. Team and Skills Required

### Phase-by-phase team build

```
Phase 0 (Weeks 1–8):   [You]
Phase 1 (Months 3–6):  [You] + [FIC Quant] + [ML Engineer]
Phase 2 (Months 7–12): Above + [Compliance] + [Backend Eng]
Phase 3 (Months 13–18): Above + [ML Eng #2] + [Frontend/UX] + [Client Success]
```

### Role definitions

| Role | What they do in dbFlow | Must-have skills |
|------|----------------------|-----------------|
| **You — Platform Architect + Product Lead** | Architecture decisions, compliance strategy, client demo, roadmap | Iceberg/Trino/Nessie/Airflow, Ollama/Gemma/RAG, product thinking |
| **FIC Quant Analyst** | Validates signal quality, defines what "good answer" means, translates trader needs | FX/rates/credit domain knowledge, Python for data analysis |
| **ML Engineer** | LoRA fine-tuning pipeline, inference optimization, model evaluation | Hugging Face PEFT, PyTorch, GKE job orchestration |
| **Compliance Specialist** | MiFID II data classification, external sharing approval, legal framework | DB data governance process, MiFID II Article 27, data licensing |
| **Backend Engineer** | FastAPI API gateway, OAuth2, rate limiting, Python SDK packaging | FastAPI, GKE, Redis (rate limiting), Python packaging |
| **Frontend / UX** | Self-service portal, client dashboard, query interface | React, TypeScript, WebSocket streaming |
| **Client Success Manager** | Alpha client onboarding, feedback collection, renewal conversations | Financial services domain, client relationship management |

### Skills you personally need to demonstrate depth in

When the room fills with senior people and they probe you, these are the domains you must answer without hesitation:

**Data platform depth:**
- Iceberg time-travel and why it matters for point-in-time correct ML training
- Nessie catalog branching and how it enables reproducible model training
- Trino federation and why you never move data between systems

**LLM/ML depth:**
- Difference between fine-tuning and RAG — when to use which
- LoRA rank selection and its effect on model adaptation vs catastrophic forgetting
- How pgvector HNSW indexing works and why it beats exact nearest-neighbour at scale
- Hallucination measurement: RAGAS framework (faithfulness, answer relevancy, context recall)

**Product depth:**
- MiFID II Article 27 best execution reporting — what data classification levels apply
- Why aggregated signals at the notional-band level pass classification but raw notional does not
- The Goldman Marquee business model: relationship deepener, not standalone product

---

## 7. Pricing and Revenue Model

### Market benchmarks

| Product | Price/yr | Data type | Exclusivity |
|---------|---------|-----------|-------------|
| Bloomberg Terminal | $28–32k/seat | Public market data | None — same for all |
| Refinitiv LSEG | $3.6–22k/seat | Public market data | None |
| Alt data (avg HF) | $1.6M total/yr | 20 datasets avg | Varies |
| Goldman Marquee | Not public | GS flow + analytics | GS clients only |
| **dbFlow (target)** | **$50k–$5M/yr** | **Real-money DB flows** | **Exclusive to DB** |

### Three-tier pricing structure

#### Tier 1 — Signal ($50k–$150k/yr per client firm)
- Weekly curated flow reports (PDF + structured JSON)
- Natural language query interface (100 queries/day)
- 3 asset class modules (choose from FX, rates, credit, trade finance)
- Target: boutique hedge funds, family offices, regional asset managers
- Onboarding: self-service within 2 business days

#### Tier 2 — Intelligence ($200k–$500k/yr per client firm) ← *highest volume tier*
- Real-time API access (unlimited queries)
- All asset class modules
- WebSocket streaming
- Anomaly alerts (push-based)
- Python SDK
- SLA: 99.5% uptime, <5s P95 latency
- Target: mid-to-large hedge funds, corporate treasury teams, SWF sub-teams

#### Tier 3 — Bespoke ($1M–$5M/yr, negotiated)
- Custom LoRA model fine-tuned on flow slices most relevant to the client's strategy
- Dedicated GKE namespace and inference endpoint
- 99.9% SLA with dedicated support
- White-glove quarterly strategy reviews
- Target: major sovereign wealth funds, large systematic hedge funds

### Revenue projections

| Year | Client mix | ARR |
|------|-----------|-----|
| Year 1 | 10× Tier 1 + 5× Tier 2 | ~€3.5M |
| Year 2 | 30× Tier 1 + 20× Tier 2 + 3× Tier 3 | ~€20M |
| Year 3 | Scaling to Goldman Marquee comparable | €100–200M |

**Gross margin: 70–85%.** Data products have near-zero marginal cost per additional client. Infrastructure cost scales sub-linearly with client count.

### The go-to-market mechanism (no cold sales)

DB's IB client coverage teams already have relationships with every hedge fund and SWF you want to target. dbFlow is introduced as part of existing client conversations — not sold as a standalone product. Clients who use dbFlow trade more through DB because the intelligence and execution are in the same ecosystem. This is the Goldman Marquee flywheel: **data subscription revenue + relationship deepening + incremental trading revenue.**

---

## 8. Hard Questions — Answered

### "Where is this data from and is it approved?"

**Answer:** Data inventory and classification audit happens in Week 1 before a single line of application code is written. dbFlow uses only aggregated, non-attributable flow signals — same level of aggregation as DB's own published quarterly flow reports, just queryable in real time. Every signal row is classified, lineaged through Nessie, and traceable to its source system. No raw client data. No position-level PII.

### "What stops a competitor from building the same thing?"

**Answer:** The data, not the technology. Any bank can run Gemma on GKE. No one outside DB can query DB's Autobahn FX flow at tick level. No one outside DB has access to DB's prime brokerage positioning book or trade finance LC database. The moat is the dataset. The stack is how you unlock it.

### "What if the LLM hallucinates and a hedge fund makes a bad trade?"

**Answer:** Three safeguards. First: every response is grounded in cited source rows shown alongside the answer — no claim is made without a traceable signal. Second: the guardrails layer blocks any output that cannot be traced to a source signal. Third: legal framing — dbFlow is an intelligence tool, not investment advice, same as Bloomberg Terminal. The client makes the trade. dbFlow informs the research process.

### "How long before someone inside DB copies this?"

**Answer:** The data governance approval and Nessie catalog lineage framework take months to establish. The LoRA-tuned model is trained on DB's proprietary internal research corpus — that corpus is not accessible to a competing internal team without going through the same governance process. First-mover advantage inside a regulated institution is stickier than in an open market.

### "What is the single biggest risk?"

**Answer:** MiFID II data classification approval taking longer than expected. Mitigation: start compliance discovery in Week 1, not Week 8. Build Phase 0 entirely on pre-approved, already-published aggregated data. Never wait for compliance to "finish" before building — expand the data envelope incrementally as each approval comes through.

### "What is your role when this gets big?"

**Answer:** Chief Architect and Product Lead. The person who understands all four layers simultaneously — the data platform (Iceberg/Trino/Nessie), the LLM infrastructure (Ollama/Gemma/RAG/LoRA), the product thinking (pricing, tiers, client workflow), and the regulatory constraints (MiFID II, data classification) — is irreplaceable in the first 18 months. As the team grows, I define the technical direction, maintain architecture integrity, and own the client-facing product roadmap.

---

## 9. My Role and Positioning

### Why I am the right person to lead this

| Domain | My evidence |
|--------|------------|
| Data platform architecture | Single-handedly architected and delivered the full Iceberg/Trino/Nessie/Airflow migration on GKE |
| LLM/AI infrastructure | Built the Ollama/Gemma RCA and anomaly detection system, approved bank-wide by the Central Architecture Board |
| Compliance-aware engineering | Operated in DB's regulated environment for years; understand data classification, air-gap constraints, and governance requirements |
| Product thinking | Designed and pitched dbFlow concept; built pricing model, tier structure, and go-to-market narrative |
| Community credibility | AWS Community Builder; external technical voice with practitioner credibility |

### The narrative arc for senior stakeholders

> "I have already built the two foundational components of dbFlow inside DB. The data platform — Iceberg, Trino, Nessie, Airflow on GKE — is running in production. The LLM infrastructure — Ollama, Gemma, air-gapped on GKE — is Architecture Board approved. dbFlow is not a new idea requiring new investment. It is a **product layer on top of proven foundations** that I can demo in 8 weeks with zero incremental infrastructure cost."

---

## 10. First Step — Week 1 Action Plan

### Monday, Week 1

**Morning (2 hours):** Pull the Nessie catalog and list every Iceberg table in the data platform. Tag by: source system, data owner, classification level, last updated. This is your working data inventory.

**Afternoon (2 hours):** Meet with the DB Data Governance contact (or Data Office). Present the concept in one sentence: "I want to build a query layer over aggregated flow signals. Here are the candidate tables. Which ones are already cleared for internal use at what aggregation level?" Get this on the record in writing.

### Tuesday–Wednesday, Week 1

Run Trino queries against the candidate tables. Document: schema, row count, freshness, null rates, obvious PII fields. Build the data quality baseline. Identify the 3–5 tables that have the cleanest data and the clearest classification status. These become your Phase 0 feature store.

### Thursday, Week 1

Design the Iceberg feature store schema (see Section 4). One table per asset class. Notional bands, not raw notionals. Flow scores, not raw volumes. Write the schema DDL. Get one domain expert (a FIC quant or a senior rates analyst) to review the signal definitions. **Do not build the feature store until a domain expert confirms the signals make sense.** Wrong signals, however cleanly ingested, produce worthless answers.

### Friday, Week 1

Write a one-page internal brief: what dbFlow is, what data it uses (with classification status), what the Phase 0 deliverable is (8-week demo), and what approvals you are seeking. Send to your direct manager and the Data Governance contact. Start the approval clock. Every week of delay in getting formal sign-off is a week of Phase 0 you cannot recover.

### Week 2–8 focus

Once data is approved:
- Week 2: Airflow DAG for daily feature store population
- Week 3: pgvector setup, research corpus embedding, LangChain RAG chain
- Week 4: Guardrails layer (query classifier + output scanner)
- Week 5–6: FastAPI endpoint, streaming responses
- Week 7: React demo interface (3 canned scenarios)
- Week 8: Demo rehearsal, edge case hardening, Innovation World Cup submission

---

## Appendix — Key Technology Decisions Summary

| Decision | Choice | Alternative considered | Reason |
|----------|--------|----------------------|--------|
| Base LLM | Gemma 2 9B via Ollama | GPT-4o, Claude API | Air-gap compliance. Already Architecture Board approved. |
| Fine-tuning approach | LoRA (PEFT) | Full fine-tune, prompt engineering only | Balance of adaptation quality vs compute cost vs catastrophic forgetting risk |
| Vector store | pgvector | Pinecone, Weaviate, Chroma | Runs inside GKE. No external data egress. Compliance requirement. |
| Feature store format | Apache Iceberg | Delta Lake, Hudi | Already the standard on the DB data platform. Time-travel is essential for reproducible ML training. |
| API framework | FastAPI | Flask, Django | Auto-generates OpenAPI spec. Async-native for streaming. Pydantic validation for structured responses. |
| Catalog versioning | Nessie | Hive Metastore, Unity Catalog | Already running. Git-like branching enables reproducible training runs with exact data versions. |
| Orchestration | Airflow on GKE | Prefect, Dagster | Already running in production. KubernetesPodOperator for isolation and auditability. |

---

*Document version: 1.0 | April 2026*  
*Classification: Internal — Innovation World Cup submission support material*  
*Next review: After Phase 0 demo completion*
