# dbFlow Demo — 2-Week Build Plan
## From zero to live Innovation World Cup demo

> **Goal:** A working, impressive, locally-runnable demo by end of Week 2.  
> **Stack:** Python 3.11 · FastAPI · Ollama + Gemma 2 9B · sentence-transformers · React frontend  
> **Hardware:** Any MacBook Pro or Linux workstation. No GPU needed for inference at demo scale.  
> **Air-gap:** Ollama runs entirely locally. No data leaves the machine during the demo.

---

## Prerequisites (before Day 1)

Install these before Week 1 starts:

```bash
# 1. Ollama (local LLM runtime)
# Mac:
brew install ollama
# Linux:
curl -fsSL https://ollama.com/install.sh | sh

# 2. Pull Gemma 2 9B (5.5GB — do this on good WiFi the night before)
ollama pull gemma2:9b

# 3. Python 3.11+
python3 --version   # should be 3.11 or higher

# 4. Node 18+ (for frontend dev server, optional)
node --version
```

---

## Week 1 — Backend + Data Layer

### Day 1 (Monday) — Project scaffold + health check

**Morning (2h):**
```bash
git clone <your-repo> dbflow-demo
cd dbflow-demo
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

Start Ollama:
```bash
ollama serve &
```

Start the API:
```bash
uvicorn app.main:app --reload --port 8000
```

Hit the health endpoint:
```bash
curl http://localhost:8000/health/
# Expected: {"status": "ok", "ollama": "connected", "model": "gemma2:9b"}
```

**Afternoon (2h):** Read through every file in the scaffold. Understand the data flow:
```
QueryRequest → guardrails.check_query() → signals.get_signals() 
→ vector_store.retrieve() → llm.chain.stream_query() → SSE stream
```

Write one test:
```bash
# Test the guardrails — this should return 400
curl -X POST http://localhost:8000/v1/query/stream \
  -H "Content-Type: application/json" \
  -d '{"query": "which client is selling TRY?"}'
```

**Day 1 done when:** Health check returns `"ollama": "connected"` and the guardrails block the test query.

---

### Day 2 (Tuesday) — First real LLM query

**Morning (2h):** Test the full pipeline end-to-end with curl:

```bash
curl -X POST http://localhost:8000/v1/query/stream \
  -H "Content-Type: application/json" \
  -N \
  -d '{
    "query": "What is the real-money positioning in EM FX this week?",
    "asset_classes": ["FX"],
    "regions": ["EMEA", "APAC"],
    "stream": true
  }'
```

You should see SSE events streaming — first `signals`, then `token` events one by one, then `sources`, then `done`.

**If Gemma is too slow** (>30 seconds first token): switch to the smaller model while developing:
```bash
ollama pull gemma2:2b
# In .env:
DBFLOW_OLLAMA_MODEL=gemma2:2b
```
Switch back to 9B before the demo.

**Afternoon (3h):** Quality check. For each of the four demo scenarios, run the query and evaluate:
- Does the first sentence correctly state the directional finding?
- Are specific signals referenced by region and date?
- Does it avoid any raw notionals or client names?
- Is the confidence statement at the end?

For each failure: adjust `SYSTEM_PROMPT` in `app/llm/chain.py`. The system prompt is your most powerful lever. Iterate on it until all four scenarios pass.

**Day 2 done when:** All four demo queries return coherent, grounded, citation-aware answers.

---

### Day 3 (Wednesday) — Vector store quality

**Goal:** Confirm that retrieval is returning the right research documents for each query.

Add a debug endpoint temporarily:
```python
# Add to app/api/signals.py
@router.get("/debug/retrieve")
async def debug_retrieve(q: str):
    from app.core.vector_store import retrieve
    docs = retrieve(q, top_k=3)
    return {"query": q, "retrieved": [d["source_title"] for d in docs]}
```

Test each scenario:
```bash
curl "http://localhost:8000/v1/signals/debug/retrieve?q=EM+FX+real+money+positioning"
curl "http://localhost:8000/v1/signals/debug/retrieve?q=hedge+fund+european+credit"
curl "http://localhost:8000/v1/signals/debug/retrieve?q=APAC+trade+finance+Q1"
curl "http://localhost:8000/v1/signals/debug/retrieve?q=USD+CNH+positioning"
```

Each should return the correct research document as the top result. If not: add more specific vocabulary to the corresponding `DEMO_CORPUS` entry in `app/core/vector_store.py`.

**Day 3 done when:** All four queries retrieve the correct top document.

---

### Day 4 (Thursday) — Signal tuning

**Goal:** Make the signal rows underneath the narration look exactly right for the demo.

Open `app/signals/store.py`. For each of the four demo scenarios, verify:
- Signal dates are recent (within the last 7 days)
- `flow_score` values are directionally consistent with the narration
- `confidence` values reflect real uncertainty (don't make everything 95%+)
- `notional_band` is always LARGE, MEDIUM, or SMALL — never a number

Add one edge case: a query that returns `confidence < 0.75`. Verify that the LLM's narration includes an explicit uncertainty caveat. If it doesn't: add this to the system prompt:

```
If any signal in the context has confidence below 0.75, explicitly 
state "Confidence on this reading is moderate" and explain why 
(e.g. thin flow data on certain days).
```

**Day 4 done when:** The USD/CNH scenario (confidence 0.78) correctly shows a caveat in the narration.

---

### Day 5 (Friday) — Error handling + demo stability

This day is about making the demo bulletproof. Murphy's law applies to live demos.

**Test every failure mode:**

```bash
# 1. Ollama is not running
pkill ollama
curl -X POST http://localhost:8000/v1/query/stream -d '{"query":"test"}'
# Should return 500 with a clean error message, not a Python traceback

# 2. Empty query
curl -X POST http://localhost:8000/v1/query/stream \
  -d '{"query":""}' -H "Content-Type: application/json"
# Should return 422 (Pydantic validation)

# 3. Guardrails blocked query
curl -X POST http://localhost:8000/v1/query/stream \
  -d '{"query":"which client is buying TRY?"}' -H "Content-Type: application/json"
# Should return 400 with reason message

# Restart Ollama for next tests
ollama serve &
```

**Build the demo backup:** Screenshot or record the response for each of the four scenarios. Save as PNG files. If Ollama is slow or crashes during the live demo, you show the screenshot and narrate.

**Day 5 done when:** All error cases return clean messages. Backup screenshots saved.

---

## Week 2 — Frontend + Polish + Rehearsal

### Day 6 (Monday) — Wire frontend to real backend

The React frontend in `frontend/index.html` currently uses hardcoded responses. Wire it to the real backend SSE stream.

Replace the `sendQuery()` function's mock section:

```javascript
// Replace the sleep(1400) + hardcoded response block with:
const evtSource = new EventSource(`/v1/query/stream?` + new URLSearchParams({
    query: q,
    asset_classes: selectedAssetClasses.join(','),
    regions: selectedRegions.join(','),
}));
// But since POST + SSE doesn't work natively, use fetch with ReadableStream:

const response = await fetch('/v1/query/stream', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({
        query: q,
        asset_classes: ['FX','CREDIT','RATES','TRADE_FINANCE'],
        regions: ['EMEA','APAC','Americas'],
        stream: true
    })
});

const reader = response.body.getReader();
const decoder = new TextDecoder();
while (true) {
    const {done, value} = await reader.read();
    if (done) break;
    const lines = decoder.decode(value).split('\n');
    for (const line of lines) {
        if (!line.startsWith('data:')) continue;
        const payload = line.slice(5).trim();
        if (payload === '[DONE]') break;
        const event = JSON.parse(payload);
        if (event.type === 'signals') renderSignals(event.signals);
        if (event.type === 'token') appendToken(event.content);
        if (event.type === 'sources') renderSources(event.sources);
    }
}
```

**Day 6 done when:** All four demo scenarios run end-to-end with real Ollama responses streaming live.

---

### Day 7 (Tuesday) — Latency optimization

Target: first token visible within **3 seconds** of clicking "Run query". This is the number judges notice.

**Profile:**
```bash
time curl -X POST http://localhost:8000/v1/query/stream \
  -d '{"query":"EM FX positioning","stream":true}' \
  -H "Content-Type: application/json" -N 2>&1 | head -5
```

**If >5 seconds to first token:**

Option 1 — use gemma2:2b for the demo (much faster, slightly lower quality):
```bash
ollama pull gemma2:2b
DBFLOW_OLLAMA_MODEL=gemma2:2b uvicorn app.main:app --reload
```

Option 2 — reduce `num_predict` from 600 to 350 in `app/llm/chain.py`. Shorter responses stream faster.

Option 3 — pre-warm Ollama by sending a dummy query on startup:
```python
# In app/main.py lifespan function, after init_vector_store():
import httpx
async with httpx.AsyncClient() as c:
    await c.post(f"{settings.OLLAMA_URL}/api/generate",
        json={"model": settings.OLLAMA_MODEL, "prompt": "Hello", "stream": False})
```

**Day 7 done when:** First token appears within 3 seconds. Full response within 15 seconds.

---

### Day 8 (Wednesday) — Demo script rehearsal

Run through the full demo three times with a timer. Hit all four scenarios.

**Timing targets:**
- Welcome + context explanation: 30 seconds
- Scenario 1 (EM FX): query + narration read-through + signal callout: 45 seconds
- Scenario 2 (HF credit): 45 seconds
- Scenario 3 (trade finance): 30 seconds
- Scenario 4 (USD/CNH — lower confidence): 30 seconds, specifically call out the caveat

**What to say while streaming:**

*[Query submitted, waiting for first token:]*
> "Notice the system is retrieving from Autobahn FX flow data first — you can see the signal rows load before the narration even starts."

*[Signals appear, narration starts streaming:]*
> "The flow score here is −0.61 — that's on a normalised scale of −1 to +1. The negative reading tells us real money is a net seller. Watch the narration reference the specific regions and counterparty sectors."

*[Narration complete:]*
> "Below the narration you see the cited source rows — every claim is traceable. This is not a hallucination — it's a grounded answer. The guardrails layer verifies this before it reaches you."

---

### Day 9 (Thursday) — Hardening

**Run the full demo 5 times without touching anything.** Count failures.

Fix every failure you see. Common issues:
- Ollama model evicted from memory between runs (fix: keep a lightweight keepalive ping)
- Vector store re-initialising too slowly (fix: cache embeddings to disk after first run)
- Frontend SSE reader hanging if Ollama is slow (fix: add a 45-second timeout with graceful error message)

**Backup mode:** Verify the static screenshot fallback is ready. Know exactly which key to press to switch to it.

---

### Day 10 (Friday) — Final rehearsal + submission prep

**Morning:** One complete run-through with someone watching. Ask them to ask one question you didn't prepare for. Answer it.

**Afternoon:** Package everything.

```bash
# Create the submission bundle
tar -czf dbflow-demo-v1.tar.gz \
  app/ frontend/ scripts/ requirements.txt README.md \
  --exclude='.venv' --exclude='__pycache__' --exclude='.git'

# Verify it runs from scratch
mkdir /tmp/dbflow-test && cd /tmp/dbflow-test
tar -xzf ~/dbflow-demo-v1.tar.gz
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --port 8001
```

---

## Architecture Decision Record

These are the decisions you defend in the Q&A.

| Decision | Choice | Why not the alternative |
|----------|--------|------------------------|
| LLM runtime | Ollama | Air-gap compliant. Runs locally. Zero external API calls during demo. Alternative (OpenAI API) would disqualify for data compliance reasons. |
| Model | Gemma 2 9B | Architecture Board approved at DB. 9B is the sweet spot: good financial language, runs on CPU at acceptable speed. 27B is too slow locally. |
| Retrieval | sentence-transformers (CPU) | No GPU needed. 84MB model. Runs in <100ms on CPU. Alternative (OpenAI embeddings) requires external API. |
| Streaming | Server-Sent Events | Native browser support. No WebSocket handshake complexity. Simpler to demo. |
| Signal store | Simulated (demo) / Iceberg (production) | Demo needs to run without a real Trino cluster. The interface to the signal store is identical — swap one function in signals/store.py to connect to real data. |
| Guardrails | Rule-based regex | Fast, deterministic, explainable. For production: replace pre-generation classifier with a fine-tuned Gemma adapter trained on blocked/allowed query pairs. |

---

## The One Thing That Must Work

If everything else breaks but this works, the demo succeeds:

1. Open the frontend
2. Click "EM FX real-money positioning"
3. Signal rows appear in under 3 seconds
4. Narration streams within 5 seconds
5. The word "Autobahn" appears in the narration (shows data provenance)
6. A cited source row appears below the narration

Everything else is polish.

---

*Document version: 1.0 | April 2026*  
*Classification: Internal · dbFlow Innovation World Cup*
