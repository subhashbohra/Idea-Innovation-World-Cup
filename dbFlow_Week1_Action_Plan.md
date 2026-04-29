# dbFlow — Week 1 Action Plan
## The First 5 Days After Approval

> **Purpose:** This document tells you exactly what to do, in what order, on each day of Week 1.  
> **Principle:** Zero application code this week. Every action is discovery, classification, or relationship-building.  
> **Output by Friday EOD:** A one-page internal brief with approved data assets, a signed-off schema sketch, and two people who are now invested in the project's success.

---

## Why Week 1 Is Different From Every Other Week

Most engineers get approval and immediately open VS Code. That is the fastest way to build the wrong thing and then spend month 3 rebuilding it after compliance says no.

Week 1 has one job: **eliminate the risks that can kill the project before it starts.**

There are exactly three of them:

1. **Data risk** — you build on data that turns out to be non-shareable externally. You discover this in month 7 when you're ready to onboard your first external client.
2. **Domain risk** — you build flow signals that your FIC quant looks at and says "this is not how traders think about this." You discover this in month 4 when your internal pilot users stop logging in.
3. **Stakeholder risk** — you build in isolation. You discover in month 6 that the person who controls the data access or the client introductions has no idea what dbFlow is and no reason to help you.

Week 1 closes all three gaps before you write a single line of code.

---

## Monday — Data Inventory

**Start time:** First thing. Before email, before Slack.

### Morning: Pull the catalog (2 hours)

Open a Trino session. Run this query first:

```sql
SHOW SCHEMAS IN nessie;
```

Then for each schema that could contain flow data:

```sql
SHOW TABLES IN nessie.<schema_name>;
```

You are looking for tables in these source domains:
- FX and rates (Autobahn-origin data)
- Trade finance (Corporate Bank — letters of credit, supply chain finance)
- Prime brokerage (hedge fund positioning, leverage, sector exposure)
- Correspondent banking (cross-border payment flows)
- Any existing aggregated flow reports that are already published internally

For each candidate table, run:

```sql
DESCRIBE nessie.<schema>.<table>;
SELECT COUNT(*) FROM nessie.<schema>.<table>;
SELECT * FROM nessie.<schema>.<table> LIMIT 5;
```

**Document in a spreadsheet — one row per table:**

| Table name | Schema | Row count | Date range | Owner team | Obvious PII? | Published externally? | Notes |
|------------|--------|-----------|------------|------------|--------------|----------------------|-------|

Target: identify 8–12 candidate tables by noon. You will not use all of them. You need to know they exist before you decide.

### Afternoon: Tag by classification risk (2 hours)

Go back through your spreadsheet. For each table, make a judgment call on one question:

**"Would publishing an aggregated, non-attributable version of this data tell a sophisticated external analyst something they could not already get from public sources?"**

If yes — it has signal value and it needs classification review.  
If no — it is not useful for dbFlow. Remove it from the list.

Then tag each remaining table:

- **GREEN** — already published in DB's quarterly flow reports or research notes at equivalent aggregation. Almost certainly pre-approved. Use in Phase 0.
- **AMBER** — useful signal, more granular than published data. Needs Data Governance review before external use. Use in Phase 1 after approval.
- **RED** — position-level, client-attributable, or restricted. Do not use. Do not even design signals around it.

**End of Monday target:** A spreadsheet with 5–8 GREEN tables and 3–5 AMBER tables. This is your data universe for the project.

---

## Tuesday — Domain Validation

**The risk you're closing:** building technically correct but commercially useless signals.

### Morning: Find your domain expert (1 hour)

You need one person from the FIC desk — a senior rates or FX analyst, or a trader who has been at DB for 5+ years — to spend 90 minutes with you this week. Not a formal meeting. A working session.

Send this message today:

> "Hi [name] — I'm working on a data intelligence project that would make DB's flow data queryable for external clients (hedge funds, SWFs). I'd love 90 minutes with you this week to sense-check whether the signals I'm designing actually match how traders think about flows. Would Thursday afternoon work? I'll bring the schema sketches."

You want someone who will tell you "that's not how we think about it" rather than someone who will politely agree with everything. Pick accordingly.

### Afternoon: Read everything that already exists (3 hours)

Before you design a single signal, read what DB already publishes:

1. **DB's quarterly FX flow reports** — what data is already disclosed externally at what aggregation level? This is your compliance baseline. If DB publishes it, it is pre-approved.
2. **DB Research — any flow-oriented publications** — how do your analysts describe directional positioning? What terminology do they use? (This is your fine-tuning corpus vocabulary.)
3. **Any existing internal dashboards or reports** that aggregate flow data — these tell you what internal consumers already find valuable.

**Take notes on:**
- Aggregation levels used (notional bands vs percentiles vs directional scores)
- Regional breakdowns (EMEA / APAC / Americas, or more granular?)
- Counterparty sector taxonomy (Hedge Fund / Real Money / Corporate / Official Sector?)
- Time granularity (daily? weekly? monthly?)
- Which asset classes are most discussed

This shapes your feature store schema. You should not design it from first principles — you should design it to match how DB already describes flows internally.

---

## Wednesday — Compliance Meeting

**This is the most important meeting of the week. Do not reschedule it.**

### Preparation (morning, 1 hour)

Prepare a one-page document to bring into the meeting. It contains:

1. **What dbFlow is** — one paragraph, plain English, no jargon
2. **The candidate data tables** — your GREEN and AMBER lists from Monday
3. **The proposed aggregation approach** — notional bands (SMALL/MEDIUM/LARGE), not raw notionals; flow scores (-1.0 to +1.0), not raw volumes; sector-level, not client-level
4. **The use case** — external institutional clients (hedge funds, SWFs, corporate treasuries) querying aggregated signals via API
5. **The specific question you need answered** — "Which of these tables, at this aggregation level, are cleared for use in an external-facing product?"

### The meeting (afternoon, 60–90 minutes)

Who you need in the room:
- Data Governance lead or Data Office representative
- Ideally: someone from Legal or Compliance who covers data licensing

**Do not go in asking for permission to build dbFlow.** Go in asking a narrower, more answerable question: *"I am using aggregated flow signals at this specific level. Which of these data sources is already cleared at this aggregation, and which needs a new review?"*

Narrower questions get faster answers. "Can I build a product on our data?" takes months. "Is this aggregated FX flow table at notional-band level already cleared?" takes days.

**What you need to leave with:**

1. Written confirmation (email is fine) of which GREEN tables are pre-approved for internal use at the described aggregation level
2. A named person and a timeline for the AMBER table reviews
3. An understanding of what the external data licensing framework looks like — do you need a data sharing agreement? A specific legal wrapper? Who owns that process?

**What you do NOT need from this meeting:**

- Full external licensing approval (that takes months — you don't need it for Phase 0)
- A legal opinion on the product (too early)
- Budget approval (wrong meeting)

**After the meeting:** Send a follow-up email summarising what was agreed. Get the approvals in writing, even informally. This protects you and the project.

---

## Thursday — Schema Design + Domain Expert Session

### Morning: Draft the feature store schema (2 hours)

Using your Tuesday research and Monday data inventory, draft the Iceberg table schemas. One table per asset class to start.

**FX flows table:**

```sql
CREATE TABLE dbflow.signals.fx_flows (
    signal_date         DATE,
    region              VARCHAR,   -- 'EMEA', 'APAC', 'Americas'
    currency_pair       VARCHAR,   -- 'USD/EM', 'EUR/USD', etc. — at pair level, not cross
    direction           VARCHAR,   -- 'NET_BUY', 'NET_SELL', 'NEUTRAL'
    counterparty_sector VARCHAR,   -- 'HEDGE_FUND', 'REAL_MONEY', 'CORPORATE', 'OFFICIAL'
    notional_band       VARCHAR,   -- 'LARGE' (>$500M), 'MEDIUM', 'SMALL'
    flow_score          DOUBLE,    -- normalised -1.0 (strong sell) to +1.0 (strong buy)
    confidence          DOUBLE,    -- 0.0 to 1.0 — how representative is the sample?
    data_source         VARCHAR,   -- which approved source table this aggregates from
    updated_at          TIMESTAMP
)
PARTITIONED BY (signal_date, region);
```

**Trade finance flows table:**

```sql
CREATE TABLE dbflow.signals.trade_finance_flows (
    signal_date         DATE,
    region              VARCHAR,
    instrument_type     VARCHAR,   -- 'LC', 'SCF', 'GUARANTEES'
    sector              VARCHAR,   -- 'COMMODITIES', 'MANUFACTURING', 'TECHNOLOGY'
    flow_direction      VARCHAR,   -- 'ISSUANCE_UP', 'ISSUANCE_DOWN', 'STABLE'
    volume_band         VARCHAR,
    flow_score          DOUBLE,
    confidence          DOUBLE,
    data_source         VARCHAR,
    updated_at          TIMESTAMP
)
PARTITIONED BY (signal_date, region);
```

**Design rules to follow:**
- No raw notionals anywhere. Ever.
- No counterparty names. Sector only.
- No individual client-attributable fields.
- Every table has a `data_source` field pointing to the approved source table. Lineage from day one.
- Every table has `confidence` — you need this to tell the LLM when a signal is based on thin data.

### Afternoon: Domain expert session (90 minutes)

Bring your schema sketches. Walk through them. Ask these specific questions:

1. **"Does `flow_score` from -1 to +1 make sense as a way to represent directional positioning? How would you describe it to a hedge fund client?"** — If they say "we'd call it net flow direction" or "we'd think about it as a z-score vs the 90-day average" — update your schema to match.

2. **"Which of these three asset classes — FX, trade finance, prime brokerage — would a hedge fund find most valuable on day one? Which would they use least?"** — This prioritises your Phase 0 build. Build the highest-value signals first.

3. **"If you had this system and a hedge fund PM asked you 'what are real-money accounts doing in EM FX this week?' — what would you want the system to return?"** — This is your first demo scenario, and you want it to come from a domain expert, not your imagination.

4. **"What is something this kind of system would get badly wrong if not designed carefully?"** — Their answer to this question is more valuable than any technical review you could do.

**After this session:** Update your schema based on their feedback. What they tell you in 90 minutes will save you months of building the wrong thing.

---

## Friday — Internal Brief + Stakeholder Mapping

### Morning: Write the internal brief (2 hours)

One page. This document does three things: it gets you on record, it gets others invested, and it starts the clock on approvals you need.

**Structure:**

---

**dbFlow — Internal Project Brief**  
*Author: [Your name] | Date: [Date] | Classification: Internal*

**What it is**  
A conversational intelligence layer over DB's aggregated institutional flow data. External institutional clients (hedge funds, SWFs, corporate treasuries) query DB's FX, trade finance, and prime brokerage flow signals via a natural language API. Think Goldman Marquee, built on DB's data moat.

**Current status**  
Phase 0 approved. Week 1 complete. Data inventory conducted across [N] candidate tables. Compliance meeting held with [name] on [date] — [X] tables pre-approved at described aggregation level, [Y] tables under review.

**Data foundation**  
Using the existing Iceberg/Trino/Nessie/Airflow data platform and the Architecture Board-approved Ollama/Gemma LLM infrastructure. Zero incremental infrastructure cost for Phase 0.

**Phase 0 deliverable (8 weeks)**  
A working demo: natural language query interface returning grounded, cited flow intelligence answers across FX and trade finance signals. Three live demo scenarios for IWC judges and internal stakeholders.

**What I need**  
- Data Governance: written confirmation of approved data assets (in progress — expected by [date])
- Business: three IB client coverage introductions for the Phase 2 alpha pilot
- HR/Finance: team budget approval for 2 hires by month 3 (FIC quant analyst + ML engineer)

**Timeline**  
Week 1–8: Phase 0 (solo, existing infrastructure). Month 3–6: internal pilot (+ 2 people). Month 7–12: external alpha, first paying clients. Month 13–18: productization.

**Contact:** [Your name, email, Slack handle]

---

Send this to:
- Your direct manager
- The Data Governance contact from Wednesday's meeting
- The IB coverage lead or anyone you know in client-facing roles
- The colleague who introduced dbFlow to the sponsoring committee (if applicable)

### Afternoon: Stakeholder map (1 hour)

Draw a simple map. Four columns:

| Name | Role | What they control | What you need from them | Status |
|------|------|------------------|------------------------|--------|
| [Data Governance contact] | Data Governance | Data classification approvals | GREEN table written confirmation + AMBER review timeline | Met Wed — follow-up sent |
| [FIC quant] | FX/Rates trader | Domain validation | Signal design review | Met Thu — schema updated |
| [IB Coverage lead] | Client relationships | Hedge fund introductions | 3 alpha client intros for Phase 2 | Brief sent — meeting TBD |
| [Your manager] | Budget + HR | Hire approvals | 2 headcount by month 3 | Brief sent |
| [Architecture Board contact] | Technical governance | Platform expansion approvals | Awareness of Phase 0 scope | Brief sent — no action needed yet |

This map tells you who you need to activate and when. In a bank, the people who control data, clients, and headcount are rarely the same people. You need all three lines moving in parallel.

---

## End of Week 1 — What You Should Have

By 6pm Friday, you should be able to answer every one of these questions without looking anything up:

- [ ] Which exact Iceberg tables are you using in Phase 0, and why are they pre-approved?
- [ ] What does your FX flow signal schema look like, and why did you choose notional bands over raw notionals?
- [ ] Who is your domain expert and what did they change in your design?
- [ ] Who is your Data Governance contact and when are they returning the AMBER table review?
- [ ] Who have you sent the internal brief to and who has acknowledged it?
- [ ] What are the three demo scenarios for the Phase 0 deliverable, and where did they come from?

If you can answer all six, Week 1 is done correctly.

---

## What You Do NOT Do in Week 1

These are the traps. Recognise them.

**Do not open VS Code for application code.** Exploration queries in Trino — fine. Setting up a local Ollama instance to test — fine. Writing the FastAPI app — not yet.

**Do not design signals in isolation.** Every signal definition should come from either a domain expert or an already-published DB research document. Your intuition about what hedge funds want is less valuable than 20 minutes with a senior FIC analyst.

**Do not ask for blanket data approval.** "Can I use all of DB's flow data?" takes six months and gets referred to 12 people. "Is this specific table, at this aggregation level, pre-approved?" takes a week.

**Do not build the demo before the schema is validated.** The demo is only as good as the signals. If the signals are wrong, the demo looks technically impressive and commercially useless — which is worse than a rough demo with the right signals.

**Do not work in stealth.** Every day you wait to send the internal brief is a day the people who can unblock you don't know they need to. The brief is not asking for permission. It is telling people what is happening and what you need.

---

## Week 2 Preview — What Comes Next

Once Week 1 is complete, Week 2 begins execution:

- **Monday:** Write the first Airflow DAG — the daily feature store population job. Trino query → aggregation → PII check → Iceberg write → Nessie commit → quality alert.
- **Tuesday–Wednesday:** pgvector setup. Embed the internal research corpus (quarterly flow reports, relevant research notes). Test retrieval quality on the three demo scenarios.
- **Thursday:** LangChain RAG chain. Connect the retriever to the Gemma/Ollama inference endpoint. First end-to-end query test.
- **Friday:** Measure. Does the answer the system returns match what your domain expert would say? If not — what is wrong: the signal, the retrieval, or the generation? Identify and fix the biggest gap.

Week 2 is where you write code. Week 1 is where you earn the right to write the right code.

---

*Document version: 1.0 | April 2026*  
*Classification: Internal | dbFlow Project*  
*Next document: dbFlow_Week2_Build_Plan.md*
