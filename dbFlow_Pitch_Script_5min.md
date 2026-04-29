# dbFlow — 5-Minute Innovation World Cup Pitch Script
## Deutsche Bank | Data-Driven Opportunities Category

---

> **Timing guide:** Each section has a target time in brackets.  
> **Demo setup:** Browser open, dbFlow interface loaded, three queries pre-typed and ready.  
> **Slide:** One slide only — the dbFlow logo and the tagline: *"The intelligence layer on DB's data moat."*  
> **Tone:** Confident, not salesy. You are a practitioner who has already built half of this.

---

## THE HOOK [0:00–0:30]

*[Pause. Look at the room. Start slow.]*

Every morning, a portfolio manager at a $10 billion hedge fund opens Bloomberg.

She pays $32,000 a year for it. And Bloomberg gives her the same data it gives every other fund in the world — public prices, public feeds, public news.

Then she calls her Deutsche Bank coverage desk. And in that 20-minute conversation, she gets something Bloomberg cannot give her.

She gets to know what the real money is actually doing.

*[Pause.]*

**dbFlow puts that conversation into an API.**

---

## THE PROBLEM AND THE MOAT [0:30–1:30]

Deutsche Bank processes trillions of dollars in FX every single day through Autobahn. We hold positioning data across our entire prime brokerage book. We see trade finance flows across Asia Pacific, Middle East, and Europe before any public indicator captures them. We move correspondent banking flows across 150 countries.

That is not just data. That is a proprietary signal that no Bloomberg terminal, no Refinitiv feed, no alternative data vendor can replicate. Because it belongs to us.

And today, that signal lives in the heads of our sales teams, our research desks, and our coverage bankers. It gets shared in phone calls and in PDF reports that take days to produce.

Goldman Sachs saw this 10 years ago and built Marquee. Today Marquee has 70,000 active monthly users and is estimated at $400 million in annual revenue.

We have equivalent — arguably superior — data. We have the infrastructure. What we have not done yet is build the intelligence layer that makes it queryable.

That is dbFlow.

---

## THE LIVE DEMO [1:30–3:00]

*[Turn to the screen. Open the dbFlow interface.]*

Let me show you what this looks like, not as a concept — as a working system.

*[Type or click the first pre-loaded query:]*

> **"What is the real-money positioning in EM FX this week?"**

*[Let the streaming response appear. Point to the screen as it generates.]*

What you are seeing is not a Bloomberg query. This is our Autobahn FX flow data — aggregated, classified, approved — being synthesised in real time by a fine-tuned LLM running entirely inside DB's infrastructure. Air-gapped. No data leaves our perimeter.

Notice the response structure: a narrative paragraph at the top, then the underlying flow signal rows below it with their sources cited. The model cannot say anything it cannot point to. That is our guardrails layer.

*[Click the second query:]*

> **"What are hedge funds doing in European credit right now?"**

*[While it streams:]*

This is pulling from our prime brokerage positioning data. Aggregated at sector level — we never expose individual client positions. The compliance layer was designed from day one, not bolted on.

*[Click the third query:]*

> **"Show me trade finance flow trends in Asia Pacific for Q1."**

*[Point to the chart / signal rows that appear:]*

This is Corporate Bank data. FX data. Prime brokerage data. All three in one interface. One query. Five seconds.

A senior DB analyst would take two days to pull this together manually. A hedge fund quant would pay $200,000 a year to have this on their terminal.

*[Step back from the screen.]*

That is dbFlow.

---

## THE BUSINESS CASE [3:00–4:00]

The alternative data market is $2.8 billion today and growing at 27% annually. The average serious hedge fund already spends $1.6 million a year on data subscriptions across 20 datasets.

dbFlow enters that market with a structural advantage no vendor can match: exclusivity. Every other data product sells the same signal to 500 funds. dbFlow sells a signal that only Deutsche Bank can produce.

Our pricing model has three tiers.

Tier 1 — $50,000 to $150,000 per year — weekly curated reports and a query interface. Boutique funds, family offices.

Tier 2 — $200,000 to $500,000 per year — real-time API, Python SDK, streaming. Mid-to-large hedge funds and corporate treasuries. This is the volume tier.

Tier 3 — $1 million to $5 million — a custom model fine-tuned on flow slices specific to the client's strategy. Sovereign wealth funds. One client at this tier funds the entire platform.

Conservative projection: 10 Tier 1 clients and 5 Tier 2 clients in year one. That is €3.5 million ARR. By year two, €20 million. The year-three trajectory puts us in Goldman Marquee territory.

And critically — every client who uses dbFlow deepens their relationship with DB. They query our intelligence and they execute through our desks. This is not just a subscription product. It is a relationship flywheel.

---

## THE CREDIBILITY AND THE ASK [4:00–5:00]

I am not standing here with a slide deck and a concept.

I have already built the foundation.

The modern data platform this runs on — Apache Iceberg, Trino, Nessie, Airflow, on GKE — I architected and delivered that migration single-handedly. It is running in production today.

The LLM infrastructure — Ollama, Gemma, air-gapped on our GDC cluster — I built that for the RCA and anomaly detection system that the Central Architecture Board approved bank-wide last quarter.

dbFlow is not a new investment. It is a product layer on top of two systems that already exist and already work.

What I need from this room is three things:

**One** — formal sponsorship to begin the data classification audit in week one, so I can move fast without compliance becoming a blocker.

**Two** — introduction to three IB client coverage contacts who already have relationships with the hedge funds and sovereign wealth funds I want to approach for the alpha pilot.

**Three** — a team budget for two hires by month three: one FIC quant analyst who can validate the signal quality, and one ML engineer who can run the fine-tuning pipeline.

With those three things, I will have a working internal pilot in six months and a first paying external client in twelve.

*[Pause. Look at the room.]*

Deutsche Bank sits on one of the most valuable proprietary datasets in global finance.

dbFlow is how we stop giving it away in phone calls and start charging for it at scale.

Thank you.

---

## Q&A PREP — THE 8 QUESTIONS THEY WILL ASK

**Q: "Where does the data come from and is it approved for external use?"**

> Data classification and governance audit happen in week one — before I write a single line of application code. dbFlow uses only aggregated, non-attributable flow signals. The same level of aggregation as our own published quarterly flow reports. Every signal row is traceable through the Nessie catalog. No raw client data. No position-level PII. We expand the data envelope incrementally as each classification approval comes through — we never wait for compliance to finish before building.

---

**Q: "What stops someone else inside DB from copying this?"**

> The data governance approval, the Nessie catalog lineage framework, and the LoRA fine-tuning on our proprietary internal research corpus all take months to establish. The research corpus I use for fine-tuning is not accessible to a competing internal team without going through the same governance process. First-mover advantage inside a regulated institution is stickier than in an open market.

---

**Q: "What if the LLM is wrong and a hedge fund loses money on a bad trade?"**

> Three safeguards. First: every response is grounded in cited source rows — the model cannot make a claim without pointing to a traceable signal. Second: our guardrails layer blocks any output that cannot be sourced. Third: legal framing — dbFlow is an intelligence tool, not investment advice. Same legal structure Bloomberg uses. The client makes the trade. We inform the research process.

---

**Q: "Goldman Sachs has a ten-year head start. Why would a hedge fund choose dbFlow over Marquee?"**

> Goldman Marquee is only available to Goldman clients. If you trade with Goldman, you get Marquee. If you trade with Deutsche Bank, Barclays, and Goldman — you currently get nothing equivalent from DB or Barclays. dbFlow serves clients who already have a DB relationship. We are not competing with Marquee for Goldman's clients. We are building the Marquee equivalent for DB's own book.

---

**Q: "What is the biggest risk?"**

> Data classification approval taking longer than planned. Mitigation: I start the compliance discovery on day one. Phase zero runs entirely on pre-approved, already-published aggregated data — zero compliance risk. We expand the data envelope incrementally as approvals arrive. The demo you just saw was built on data that is already classified and approved.

---

**Q: "How much does it cost to build?"**

> Phase zero — the working demo — is zero incremental infrastructure cost. It runs on existing GKE clusters, existing Ollama/Gemma deployment, existing Iceberg tables. The only cost is my time for eight weeks. Phase one requires two hires. Phase two requires two more. By the time we are generating €3.5 million in annual revenue, we have spent less than €1 million building it. The gross margin on this product, at scale, is 80%.

---

**Q: "Why are you the right person to lead this?"**

> I am the only person in this organisation who currently bridges all four layers simultaneously: the data platform — I built it; the LLM infrastructure — I built it; the product design — I designed it; and the regulated environment constraints — I operate in them daily. The person who replaces me in this role does not exist as a single hire. You would need four people to cover what I already know. That is not an ego claim. That is a practical observation about what this project requires in its first eighteen months.

---

**Q: "What do you need from us today — specifically?"**

> Three things. One: formal sponsorship for the data classification audit to begin immediately. Two: three IB client coverage introductions to existing hedge fund or SWF relationships. Three: a team budget for two hires by month three — one FIC quant analyst, one ML engineer. Everything else I can build myself.

---

## TIMING CHEAT SHEET

| Section | Target | Hard stop |
|---------|--------|-----------|
| The hook | 0:30 | 0:35 |
| Problem and moat | 1:00 | 1:05 |
| Live demo | 1:30 | 1:40 |
| Business case | 1:00 | 1:05 |
| Credibility and ask | 1:00 | 1:05 |
| **Total** | **5:00** | **5:30** |

> **Practise rule:** If you cannot do the demo section in 90 seconds in rehearsal, cut one of the three demo queries. Two queries at 45 seconds each is better than three queries that run over time.

---

## DEMO SETUP CHECKLIST (night before)

- [ ] dbFlow interface loaded in browser, full screen
- [ ] Three queries pre-typed in the input field (just press Enter to run)
- [ ] Streaming responses tested — confirm < 5 second latency
- [ ] Backup: screenshot of each response if live system is unavailable
- [ ] Slide deck: one slide only — dbFlow logo + tagline
- [ ] Phone on silent, face down
- [ ] Timer app open on phone for self-monitoring

---

*Pitch script version: 1.0 | April 2026*  
*Rehearsal target: 3 full run-throughs before submission day*  
*Document classification: Internal | Innovation World Cup*
