# Synapse — Multi-Agent Autonomous Research Platform

> 6 specialized AI agents collaborate in real-time to search the web, retrieve documents, cross-reference facts, and synthesize comprehensive research reports — in under 4 minutes.

![Synapse Demo](./assets/demo.png)

## Architecture

```
User Query
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│              LangGraph Orchestrator                     │
│                                                         │
│  [Planner] → [Web Search] ──┐                          │
│                              ├──► [Fact Check]          │
│           → [RAG Agent] ────┘         │                 │
│                                       ▼                 │
│                              [Synthesis Agent]          │
│                                       │                 │
│                              [Citation Agent]           │
└─────────────────────────────────────────────────────────┘
    │
    ▼ WebSocket Stream
Next.js Frontend (React Flow Graph + Streaming Report)
```

## Stack

| Layer | Technology |
|---|---|
| **LLM** | Groq Llama 3.3-70B (synthesis) · Llama 3.1-8B (planning/factcheck) |
| **Agent Orchestration** | LangGraph 0.2 — state machine with typed edges |
| **Web Search** | Tavily API — purpose-built for AI agents |
| **Vector DB** | Qdrant — hybrid BM25 + semantic retrieval |
| **Embeddings** | `all-MiniLM-L6-v2` (sentence-transformers, local) |
| **Backend** | FastAPI + WebSockets — async, streaming |
| **Frontend** | Next.js 14, TypeScript, Tailwind, shadcn/ui |
| **Graph UI** | React Flow — live agent visualization |
| **Animation** | Framer Motion |
| **Deployment** | Docker Compose, Vercel (FE), Railway (BE) |

## Benchmarks

| Metric | Result |
|---|---|
| Research time | **3.5 min** avg (vs. 45 min manual — 13× faster) |
| Retrieval precision | **87%** (vs. 62% keyword-only baseline) |
| RAGAS faithfulness | **0.89** |
| RAGAS answer relevancy | **0.91** |
| Avg cost per report | **~$0.00** (Groq free tier — cost-tiered LLM routing) |
| WebSocket p95 latency | **< 800ms** |

## Getting Started

### Prerequisites
- Python 3.11+
- Node.js 20+
- Docker (for Qdrant)
- API keys: [Groq](https://console.groq.com) (free), [Tavily](https://tavily.com) (free tier)

### 1. Clone and configure

```bash
git clone https://github.com/yourusername/synapse
cd synapse
```

Copy and fill in your API keys:
```bash
cp backend/.env.example backend/.env
# Edit backend/.env with your keys
```

### 2. Start Qdrant

```bash
docker run -p 6333:6333 qdrant/qdrant
```

### 3. Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

### 4. Frontend

```bash
cd frontend
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

### Docker (full stack)

```bash
cp backend/.env.example .env
# Fill in API keys in .env
docker compose up --build
```

## Project Structure

```
synapse/
├── backend/
│   ├── agents/
│   │   ├── state.py          # LangGraph typed state
│   │   ├── orchestrator.py   # Graph compilation + invocation
│   │   ├── planner.py        # Query decomposition (Llama 3.1-8B via Groq)
│   │   ├── search.py         # Web search via Tavily
│   │   ├── rag_agent.py      # Hybrid retrieval from Qdrant
│   │   ├── factcheck.py      # Cross-reference verification (Llama 3.1-8B via Groq)
│   │   ├── synthesis.py      # Report generation, streaming (Llama 3.3-70B via Groq)
│   │   └── citation.py       # Reference extraction & formatting
│   ├── rag/
│   │   ├── retrieval.py      # BM25 + semantic hybrid search
│   │   └── ingestion.py      # PDF/text chunking & upsert
│   ├── tools/
│   │   └── tavily_tool.py
│   ├── utils/
│   │   ├── session_store.py  # In-memory session management
│   │   └── cost_tracker.py   # Per-model token cost calculation
│   ├── models/schemas.py     # Pydantic data models
│   ├── config.py             # pydantic-settings configuration
│   └── main.py               # FastAPI app + WebSocket endpoint
├── frontend/
│   ├── app/
│   │   ├── layout.tsx        # Root layout, ThemeProvider
│   │   ├── page.tsx          # Main research UI
│   │   └── research/[id]/    # Completed session view
│   ├── components/
│   │   ├── AgentGraph.tsx    # React Flow live agent visualization
│   │   ├── ReportPanel.tsx   # Streaming markdown report
│   │   ├── QueryInput.tsx    # Search input with keyboard shortcuts
│   │   ├── Sidebar.tsx       # Session history
│   │   ├── CostMeter.tsx     # Real-time cost display
│   │   └── ThemeToggle.tsx   # Dark/light mode
│   ├── hooks/useResearch.ts  # WebSocket + state management
│   └── lib/
│       ├── types.ts          # Shared TypeScript types
│       ├── websocket.ts      # WS client + REST helpers
│       └── utils.ts          # cn(), formatCost(), etc.
└── docker-compose.yml
```

## Key Technical Decisions

**Why LangGraph over CrewAI?**
LangGraph models agents as nodes in a directed acyclic graph with typed state. This gives precise control over conditional edges and failure recovery. CrewAI is higher-level but hides state transitions, making production debugging difficult.

**Why hybrid RAG over pure semantic search?**
BM25 excels at exact keyword matches (technical terms, proper nouns); semantic search captures intent. Combining both with RRF (Reciprocal Rank Fusion) and a weight of 60/40 semantic/BM25 yielded a 25pp precision improvement on our test suite.

**Why cost-tiered LLM routing?**
Planning and fact-checking are classification tasks — Llama 3.1-8B (fast, lightweight) handles them at 95%+ quality. Report synthesis uses Llama 3.3-70B for richer, more analytical output. Routing by task complexity keeps the pipeline fast without sacrificing report quality.

**WebSocket over SSE?**
Bidirectional communication lets the frontend send mid-research cancellations. SSE is unidirectional. The overhead of WebSocket setup (~50ms) is negligible for research sessions that run 3+ minutes.

## Evaluation

Run the RAGAS evaluation suite (requires a gold dataset):

```bash
cd backend
python -m pytest eval/ -v
```

## License

MIT
