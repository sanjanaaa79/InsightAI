# InsightAI — Multi-Agent Business Intelligence Assistant

InsightAI turns raw CSV and Excel files into cleaned data, dashboards, forecasts, business reports and a chat
you can ask questions in plain English. A set of **LangGraph** agents does the work, using a **local LLM served by
Ollama**, so your data never leaves your machine.

**Stack:** Python · FastAPI · LangGraph · LangChain · Ollama · PostgreSQL + pgvector · React (Vite) · Recharts · Docker

---

## Features

| | |
|---|---|
| **Automated data cleaning** | Standardizes column names; parses currency, %, thousand separators, negatives written as `(123)` and mixed date formats; turns placeholders (`N/A`, `-`, `null`) into real missing values; merges spelling variants of the same category (`west `, `WEST` → `West`); removes duplicates; fills missing values; flags outliers. Every step is written to an audit log. |
| **Exploratory analysis** | Detects column types (id, numeric, categorical, datetime, text), computes statistics and correlations, and gives the raw data a quality score. |
| **Auto dashboards** | KPI tiles, trend lines, category breakdowns, distributions, scatter plots and a correlation heatmap, chosen from the column roles the EDA agent detects. |
| **Forecasting** | Holt-Winters exponential smoothing (seasonal when there's enough history) with 95% prediction intervals and a back-test accuracy score (MAPE). Any date/metric pair works, with sum/mean/count and daily/weekly/monthly options. |
| **Insights & recommendations** | Statistics are computed with pandas first. The LLM only turns these verified numbers into findings, prioritized recommendations and risks, which keeps it from inventing figures. |
| **Natural-language querying (RAG + Text-to-SQL)** | Ask questions in chat. A router agent sends each question either to a SQL agent (which retries with the error message when a query fails) or to an advisor that answers from the retrieved knowledge base. Answers include a chart, a result table, the SQL used and the steps each agent took. |
| **Reports** | An executive report you can download as HTML or Markdown, or print to PDF. |
| **Works offline without the LLM** | If Ollama isn't running, rule-based insights and keyword retrieval take over, and chat accepts raw `SELECT` queries. |

## Architecture

```
                 ┌──────────────── React + Recharts (Vite, nginx) ───────────────┐
                 │ Dashboard · Insights · Ask AI · Forecast · Quality · Report   │
                 └───────────────────────────────┬───────────────────────────────┘
                                                 │ REST (/api)
┌────────────────────────────────────── FastAPI ─┴───────────────────────────────────────┐
│                                                                                        │
│  Analysis workflow (LangGraph, runs on upload)                                         │
│  ingest → Data Cleaning Agent → EDA Agent ─┬→ Forecast Agent ─┬→ Insight Agent (LLM)   │
│                                            └──(no date col)───┘         │              │
│                                  Report Agent ← Knowledge Agent (RAG indexing) ←┘       │
│                                                                                        │
│  Query workflow (LangGraph, runs per question)                                         │
│  Retriever (pgvector) → Router (LLM) ─DATA──→ SQL Agent → SQL Guard/Executor ─┐        │
│                                      │            ↑______ retry on error ______│        │
│                                      │                                         ↓        │
│                                      ├─GENERAL→ Advisor (RAG answer)      Analyst (LLM)  │
│                                      └─SELECT…→ Executor (raw SQL)                      │
└───────────────┬───────────────────────────────────────────────┬────────────────────────┘
                │                                               │
     PostgreSQL + pgvector                               Ollama (local)
     • one table per dataset (ds_<id>)                   • llama3.1:8b (chat)
     • metadata, chat history (JSONB)                    • nomic-embed-text (embeddings)
     • LangChain PGVector knowledge collections
```

### What the RAG knowledge base contains

For each dataset, the Knowledge Agent embeds the following into a pgvector collection:
- a schema overview and one data-dictionary document per column, including the **exact category spellings**, which keeps SQL filters from missing on case or spacing
- pre-computed aggregates (metric by category) and the mined EDA facts
- the generated insights and recommendations
- chunks of sample records

The SQL agent and the advisor retrieve the top-k chunks for each question. If embeddings are unavailable, a TF-IDF-style keyword ranker searches the same documents instead.

### Safety of generated SQL
Before any LLM-written query runs:
- `sqlglot` parses it, and it must be a single `SELECT`/CTE that reads only that dataset's table (no system catalogs, no other tables, no DDL or DML).
- It then runs in a `READ ONLY` transaction with a statement timeout and a row limit, and the transaction is always rolled back.

## Quick start (Docker)

```bash
cp .env.example .env          # optional: change LLM_MODEL (e.g. llama3.2:3b for low-RAM machines)
docker compose up --build
```

- App: http://localhost:3000
- API docs: http://localhost:8000/docs

The `ollama-pull` service downloads the models on first start. `llama3.1:8b` is about 4.7 GB and needs about 8 GB RAM; for a lighter setup use `LLM_MODEL=llama3.2:3b`. The app works before the download finishes and switches to LLM mode once the models are available. Click **Re-analyze** on a dataset to regenerate its insights with the LLM.

For NVIDIA GPUs, uncomment the `deploy` block of the `ollama` service in `docker-compose.yml`.

## Local development

```bash
# 1. Postgres + Ollama
docker compose up -d postgres ollama ollama-pull

# 2. Backend
cd backend
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt -r requirements-dev.txt
uvicorn app.main:app --reload                        # http://localhost:8000

# 3. Frontend
cd frontend
npm install
npm run dev                                          # http://localhost:5173 (proxies /api → :8000)

# Tests (no DB/LLM needed)
cd backend && pytest
```

## API overview

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/datasets` | Upload CSV/Excel (multipart) and start the analysis workflow |
| `POST` | `/api/datasets/sample` | Load a generated, deliberately messy retail dataset |
| `GET` | `/api/datasets` / `/api/datasets/{id}` | List datasets / full analysis (profile, dashboard, insights, forecast) |
| `GET` | `/api/datasets/{id}/preview` | Paginated cleaned rows |
| `POST` | `/api/datasets/{id}/forecast` | Custom forecast `{date_column, value_column, periods, aggregation, frequency}` |
| `GET` | `/api/datasets/{id}/report[/download?format=html\|md]` | Business report |
| `POST` | `/api/datasets/{id}/reanalyze` | Re-run all agents |
| `GET/POST/DELETE` | `/api/datasets/{id}/chat` | Conversational Q&A over the dataset |
| `GET` | `/api/health` | Database + Ollama/model status |

## Project structure

```
backend/app
├── agents/
│   ├── analysis.py      # LangGraph analysis workflow (cleaning → EDA → forecast → insights → RAG → report)
│   ├── query.py         # LangGraph Q&A workflow (retriever, router, SQL agent, executor, analyst, advisor)
│   ├── insights.py      # Insight agent (structured LLM output + rule-based fallback)
│   └── prompts.py
├── services/
│   ├── ingestion.py     # CSV/Excel loading (delimiter + encoding sniffing)
│   ├── cleaning.py      # cleaning toolkit with audit log and quality score
│   ├── profiling.py     # EDA, semantic types, column-role selection
│   ├── dashboard.py     # KPI + chart spec generation
│   ├── forecasting.py   # Holt-Winters / Holt / linear with back-testing
│   ├── facts.py         # deterministic fact mining fed to the LLM
│   ├── rag.py           # knowledge docs, pgvector indexing, retrieval
│   ├── sql_guard.py     # SQL validation + read-only execution
│   ├── llm.py           # ChatOllama / OllamaEmbeddings, LCEL helpers, health
│   └── report.py        # Markdown report + HTML export
├── routers/             # FastAPI endpoints
└── main.py
frontend/src/components  # Dashboard, Insights, Chat, Forecast, Quality, Report, Data views
```

## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `DATABASE_URL` | `postgresql+psycopg://insight:insight@localhost:5432/insightai` | Postgres (must have pgvector) |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama server |
| `LLM_MODEL` | `llama3.1:8b` | Chat model for the agents |
| `EMBED_MODEL` | `nomic-embed-text` | Embedding model for RAG |
| `ENABLE_LLM` | `true` | Set `false` to force rule-based mode |
| `SQL_ROW_LIMIT` / `SQL_TIMEOUT_MS` | `500` / `15000` | Guardrails for generated SQL |
| `MAX_UPLOAD_MB` | `50` | Upload limit |
