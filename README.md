<div align="center">

# 🧠 OpsMind

### AI-powered IT operations platform — multi-agent ticket triage, grounded in your own knowledge base

OpsMind turns every support ticket into a guided, auditable workflow: a team of cooperating AI agents diagnoses the issue, searches your runbooks, assesses security risk, proposes a fix with a rollback plan, and flags anything risky for human approval — all running on **local models**, so your data never leaves your infrastructure.

![Python](https://img.shields.io/badge/Python-3.14-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-multi--agent-1C3C3C)
![Next.js](https://img.shields.io/badge/Next.js_16-000000?logo=nextdotjs&logoColor=white)
![React](https://img.shields.io/badge/React_19-61DAFB?logo=react&logoColor=black)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-4169E1?logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-Celery-DC382D?logo=redis&logoColor=white)
![Ollama](https://img.shields.io/badge/Ollama-100%25_local_AI-black)

</div>

---

## ✨ Why OpsMind?

| | |
| --- | --- |
| 🤖 **Seven cooperating agents** | Supervisor, Diagnosis, RAG, Security, Resolution, Reviewer and Ticket agents orchestrated with LangGraph. |
| 📚 **Answers grounded in your docs** | Upload PDF, DOCX, XLSX, HTML and TXT files. They are chunked, embedded and searched with pgvector, so suggestions come from *your* runbooks. |
| 🔒 **Private by design** | LLM and embeddings run locally through Ollama. No tickets or documents are sent to third-party AI services. |
| 🛡️ **Safety built in** | Every proposed fix carries a risk rating, a step-by-step plan and a rollback plan. High-risk actions are flagged for human approval. |
| ⚡ **Real-time** | A WebSocket assistant streams agent progress live to the dashboard. |
| 🔌 **Jira-ready** | Create, update, comment on and sync Jira issues straight from OpsMind, with every action audit-logged. |

## 🚀 Features

### Intelligent ticket workflow
- **LangGraph state machine** with conditional routing, a reviewer feedback loop and a bounded recursion limit.
- **Structured outputs everywhere.** Every agent returns validated Pydantic models, not free-form text.
- **Self-review.** A reviewer agent approves a plan, or sends it back to diagnosis for another pass.
- **Smart fallbacks.** The workflow skips ahead to the ticket stage after repeated errors instead of stalling.
- **Editable prompts.** Each agent's prompt lives in a plain `.txt` file you can tune without touching code.

### Knowledge base (RAG)
- Ingests **PDF, DOCX, XLSX, HTML and TXT**.
- Configurable chunk size, overlap and top-k retrieval.
- **pgvector** similarity search in PostgreSQL, with embeddings from `bge-m3`.
- Optional **Celery** background ingestion for large files.
- Content hashing and file-size limits for safe uploads.

### Security and access control
- **JWT authentication** with **bcrypt** password hashing.
- **Role-based access control:** `employee`, `it_support`, `it_admin`.
- Audit logging for Jira operations and ticket changes.
- Consistent JSON error envelope with a **request ID** on every response (`x-request-id`) for easy tracing.
- Configurable CORS and token lifetime.

### Observability
- Per-agent **run tracking** stored in the database.
- Optional **LangSmith** tracing with workflow and ticket metadata.
- Structured logging for workflow start and completion.
- Health endpoints for the **API, PostgreSQL, Redis and Ollama**, shown live on the Monitoring page.

### Modern dashboard
- **IT command center** with ticket status and priority charts (Recharts), recent tickets and system health.
- **Tickets** with filtering, detail view, conversation thread, status management and a one-click **Run workflow** button.
- **AI assistant** with live agent status over WebSocket.
- **Documents** with upload, ingestion and a searchable knowledge base.
- **Approvals** view showing the proposed resolution, supporting evidence and rollback plan.
- **Jira** and **Admin** pages, with role-aware navigation.
- Responsive layout built with **Tailwind CSS 4**.

## 🧩 How it works

```mermaid
flowchart LR
    S([Start]) --> SUP[Supervisor<br/>categorise and prioritise]
    SUP -->|ok| DIAG[Diagnosis]
    SUP -->|2 or more errors| TKT
    DIAG --> RAG[RAG<br/>retrieve runbook context]
    RAG --> SEC[Security<br/>assess risk]
    SEC --> RES[Resolution<br/>steps, expected result, rollback]
    RES --> REV{Reviewer}
    REV -->|approved| TKT[Ticket<br/>update and persist]
    REV -->|needs human approval| TKT
    REV -->|rejected, one retry| DIAG
    TKT --> E([End])
```

| Step | Agent | What it does |
| --- | --- | --- |
| 1 | **Supervisor** | Classifies the request by category and priority. |
| 2 | **Diagnosis** | Analyses the likely root cause. |
| 3 | **RAG** | Retrieves the most relevant chunks from your uploaded documents. |
| 4 | **Security** | Rates operational risk (`low` / `medium` / `high` / `critical`) and decides whether a human must approve. |
| 5 | **Resolution** | Produces the proposed action, ordered steps, expected result and rollback plan. |
| 6 | **Reviewer** | Returns `approved`, `rejected` or `request_more_information`. |
| 7 | **Ticket** | Persists the outcome: `resolved`, or `in_progress` when approval is required. |

## 🏗️ Architecture

```
┌────────────────────┐   REST + WebSocket   ┌──────────────────────────────┐
│  Next.js frontend  │ ───────────────────▶ │        FastAPI backend       │
│  (React 19, TW 4)  │ ◀─────────────────── │                              │
└────────────────────┘                      │  routes → services → repos   │
                                            │  LangGraph agents  · RAG     │
                                            └───┬───────────┬──────────┬───┘
                                                │           │          │
                                   ┌────────────▼──┐  ┌─────▼─────┐  ┌─▼─────────┐
                                   │ PostgreSQL    │  │  Redis    │  │  Ollama   │
                                   │ + pgvector    │  │  + Celery │  │ LLM +     │
                                   │ (data, vectors│  │  workers  │  │ embeddings│
                                   │  audit log)   │  └───────────┘  └───────────┘
                                   └───────────────┘
                                            │
                                      ┌─────▼─────┐
                                      │   Jira    │  (optional)
                                      └───────────┘
```

A clean layered backend (`api` → `services` → `repositories` → `db`) keeps HTTP handling, business logic and persistence separate and easy to test.

## 🛠️ Tech stack

| Layer | Technology |
| --- | --- |
| **Backend** | Python 3.14, FastAPI, Uvicorn, Pydantic v2, SQLAlchemy 2 (async), Alembic |
| **AI / agents** | LangGraph, LangChain, Ollama (`gemma3:4b` for generation, `bge-m3` for embeddings), LangSmith (optional) |
| **Data** | PostgreSQL + pgvector (asyncpg), Redis, Celery |
| **Document parsing** | pypdf, python-docx, openpyxl, BeautifulSoup, tiktoken |
| **Auth** | JWT (python-jose), bcrypt / passlib |
| **Integrations** | Jira (atlassian-python-api) |
| **Frontend** | Next.js 16, React 19, TypeScript, Tailwind CSS 4, Axios, Recharts, lucide-react |
| **Tooling** | uv, pytest, ruff, black, mypy, ESLint |

## 📁 Project structure

```
OpsMind/
├── backend/
│   ├── alembic/                 # Database migrations (incl. pgvector setup)
│   ├── src/backend/
│   │   ├── main.py              # FastAPI app entry point
│   │   ├── api/                 # Routes: auth, users, tickets, documents, jira, workflows, ws, health
│   │   ├── agents/              # LangGraph workflow, nodes, state, run tracking
│   │   ├── ai/                  # Ollama LLM and embedding clients, schemas, prompt files
│   │   ├── rag/                 # Upload, text extraction, chunking, retrieval
│   │   ├── integrations/jira/   # Jira client and sync service
│   │   ├── services/            # Business logic
│   │   ├── repositories/        # Data access
│   │   ├── db/                  # Engine, sessions, Redis, ORM models
│   │   ├── worker/              # Celery app and tasks
│   │   ├── schemas/             # Pydantic request/response models
│   │   └── core/                # Config, security, logging, error handling
│   ├── tests/                   # pytest suite
│   ├── pyproject.toml
│   └── requirements.txt
└── frontend/
    ├── app/                     # Pages: dashboard, tickets, assistant, documents, approvals, jira, monitoring, admin
    ├── components/              # App shell and UI primitives
    └── lib/                     # API client, auth, hooks, types
```

## ⚙️ Getting started

### Prerequisites

- **Python 3.14** and [**uv**](https://docs.astral.sh/uv/)
- **Node.js 20+** and npm
- **PostgreSQL** with the [pgvector](https://github.com/pgvector/pgvector) extension available
- **Redis** (or Valkey)
- [**Ollama**](https://ollama.com)
- *(Optional)* a Jira Cloud project and API token

### 1. Clone the repository

```bash
git clone https://github.com/Tiwari-Tech/OpsMind.git
cd OpsMind
```

### 2. Pull the local models

```bash
ollama pull gemma3:4b
ollama pull bge-m3
```

### 3. Start the backend

```bash
cd backend
uv sync

# Create backend/.env (see Configuration below), then:
alembic upgrade head          # applies migrations and enables pgvector
uv run uvicorn backend.main:app --reload --port 8000
```

Interactive API docs: **http://localhost:8000/docs**

### 4. (Optional) Start the Celery worker

For background document ingestion:

```bash
cd backend
uv run celery -A backend.worker.celery_app.celery_app worker --loglevel=info
```

### 5. Start the frontend

```bash
cd frontend
npm install
npm run dev
```

Open **http://localhost:3000**, register an account and create your first ticket.

> The frontend proxies `/api/*` and `/health/*` to the backend (default `http://127.0.0.1:8000`). Change it with `NEXT_PUBLIC_API_BASE_URL`.

## 🔧 Configuration

Settings load from `backend/.env`. Only `DATABASE_URL` is required.

```env
# --- Required ---
DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/opsmind

# --- Security ---
JWT_SECRET_KEY=replace-with-a-long-random-string
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60
CORS_ORIGINS=http://localhost:3000,http://127.0.0.1:3000

# --- Redis / Celery ---
REDIS_URL=redis://localhost:6379
# CELERY_BROKER_URL=            # defaults to REDIS_URL
# CELERY_RESULT_BACKEND=        # defaults to REDIS_URL

# --- Ollama ---
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_LLM_MODEL=gemma3:4b
OLLAMA_EMBEDDING_MODEL=bge-m3
OLLAMA_TIMEOUT_SECONDS=60

# --- RAG ---
DOCUMENT_STORAGE_DIR=uploads/documents
MAX_UPLOAD_BYTES=10485760       # 10 MB
RAG_CHUNK_SIZE=1200
RAG_CHUNK_OVERLAP=200
RAG_TOP_K=5

# --- Jira (optional) ---
JIRA_BASE_URL=https://your-company.atlassian.net
JIRA_EMAIL=you@company.com
JIRA_API_TOKEN=your-token
JIRA_PROJECT_KEY=IT

# --- LangSmith tracing (optional) ---
LANGSMITH_TRACING=false
LANGSMITH_API_KEY=
LANGSMITH_PROJECT=opsmind
```

> `.env` files are git-ignored, so your secrets stay local.

## 📡 API overview

| Area | Endpoints |
| --- | --- |
| **Auth** | `POST /api/v1/auth/register` · `POST /api/v1/auth/login` · `GET /api/v1/auth/me` |
| **Users** | `GET /api/v1/users/me` · `GET /api/v1/users/{id}` |
| **Tickets** | `POST` / `GET /api/v1/tickets` · `GET` / `PATCH /api/v1/tickets/{id}` · `POST /api/v1/tickets/{id}/messages` |
| **Documents** | `POST /api/v1/documents/upload` · `POST /api/v1/documents/{id}/ingest` · `POST /api/v1/documents/{id}/ingest-task` · `GET /api/v1/documents` · `GET /api/v1/documents/{id}` |
| **Workflows** | `POST /api/v1/workflows/tickets/{ticket_id}/run` |
| **Real-time** | `WS /api/v1/ws/chat?token=<jwt>` |
| **Jira** | `POST /api/v1/jira/issues` · `GET` / `PATCH /api/v1/jira/issues/{key}` · `POST /api/v1/jira/issues/{key}/comments` · `POST /api/v1/jira/tickets/{id}/sync-status` |
| **Health** | `GET /health` · `/health/db` · `/health/redis` · `/health/ollama` |

Full schemas are available in the Swagger UI at `/docs`.

### WebSocket example

Connect to `ws://localhost:8000/api/v1/ws/chat?token=<jwt>` and send:

```json
{ "ticket_id": "3f1c2b0e-0000-0000-0000-000000000000" }
```

You'll receive live events: `connected`, `message_received`, `agent_started` and `approval_required` (with workflow ID and risk level).

## 👥 Roles

| Role | Intended for |
| --- | --- |
| `employee` | Raising and following tickets |
| `it_support` | Working tickets and using the Jira integration |
| `it_admin` | Full access, including the admin area |

## ✅ Testing and code quality

```bash
cd backend
uv run pytest          # auth, models, API routes and health checks
uv run ruff check .    # lint
uv run black .         # format
uv run mypy src        # type-check

cd ../frontend
npm run lint
```

## 👤 Author

Built by **[Tiwari-Tech](https://github.com/Tiwari-Tech)**. Feedback, issues and ideas are welcome.
