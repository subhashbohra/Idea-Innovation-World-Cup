# dbFlow Demo
### Deutsche Bank Innovation World Cup · Data-Driven Opportunities

Conversational flow intelligence platform. LLM trained on DB's institutional flow data.  
Runs entirely locally. Air-gapped. No data leaves the machine.

---

## Quick start (5 minutes)

```bash
# 1. Install Ollama and pull the model (one-time, ~10 minutes)
ollama pull gemma2:9b

# 2. Install Python dependencies
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 3. Start Ollama (if not already running)
ollama serve &

# 4. Start the API + frontend
uvicorn app.main:app --reload --port 8000

# 5. Open the demo
open http://localhost:8000
```

## Project structure

```
dbflow-demo/
├── app/
│   ├── main.py              # FastAPI app entry point
│   ├── api/
│   │   ├── query.py         # POST /v1/query/stream — SSE streaming
│   │   ├── signals.py       # GET /v1/signals/live
│   │   └── health.py        # GET /health/
│   ├── core/
│   │   ├── config.py        # All settings (override via .env)
│   │   ├── models.py        # Pydantic models — the API contract
│   │   ├── guardrails.py    # Pre/post-generation safety layer
│   │   └── vector_store.py  # In-memory RAG (replace with pgvector in prod)
│   ├── signals/
│   │   └── store.py         # Simulated signals (replace with Trino in prod)
│   └── llm/
│       └── chain.py         # RAG pipeline → Ollama streaming
├── frontend/
│   └── index.html           # React demo interface
├── scripts/
│   └── setup.sh             # One-command setup
├── requirements.txt
├── DEMO_BUILD_PLAN.md       # Day-by-day 2-week build guide
└── README.md
```

## Switching to real data (production path)

**Replace simulated signals with real Iceberg/Trino:**
```python
# In app/signals/store.py, replace get_signals() with:
import trino
conn = trino.dbapi.connect(host="your-trino-host", port=8080, catalog="nessie")
cur = conn.cursor()
cur.execute("SELECT ... FROM nessie.dbflow_signals.fx_flows WHERE ...")
```

**Replace in-memory vector store with pgvector:**
```bash
DBFLOW_VECTOR_STORE=pgvector
DBFLOW_PGVECTOR_URL=postgresql://localhost:5432/dbflow
```

**Switch to fine-tuned model:**
```bash
# After LoRA fine-tuning:
ollama create dbflow-gemma --file ./Modelfile
DBFLOW_OLLAMA_MODEL=dbflow-gemma
```

## API reference

```bash
# Health
GET  /health/

# Live signals (sidebar)  
GET  /v1/signals/live

# Streaming query (main endpoint)
POST /v1/query/stream
{
  "query": "What is EM FX positioning this week?",
  "asset_classes": ["FX", "CREDIT"],
  "regions": ["EMEA", "APAC"],
  "date_range_days": 7,
  "stream": true
}

# Interactive API docs
GET  /docs
```

---

*dbFlow · Innovation World Cup 2025 · Deutsche Bank*  
*Classification: Internal · Demo only · Not for production use*
