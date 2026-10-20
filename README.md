# OpsMind

**An AI-powered IT operations platform that triages support tickets with a team of cooperating LLM agents, grounds every answer in your own runbooks, and keeps a human in the loop for anything risky.**

OpsMind pairs a **FastAPI + LangGraph** backend with a **Next.js** dashboard. Employees raise tickets, a multi-agent workflow diagnoses the problem, retrieves relevant knowledge-base content (RAG), assesses security risk, proposes a resolution with a rollback plan, and has a reviewer agent sign off. High-risk actions are flagged for human approval. Everything runs on **local models through Ollama**, so your tickets and documents never leave your infrastructure.

![Python](https://img.shields.io/badge/Python-3.14-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-1C3C3C)
![Next.js](https://img.shields.io/badge/Next.js_16-000000?logo=nextdotjs&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-4169E1?logo=postgresql&logoColor=white)
![Ollama](https://img.shields.io/badge/Ollama-local_LLMs-black)

---

## Table of contents

- [Features](#features)
- [How it works](#how-it-works)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Getting started](#getting-started)
- [Configuration](#configuration)
- [API overview](#api-overview)
- [Roles and security](#roles-and-security)
- [Testing and code quality](#testing-and-code-quality)
- [Project status](#project-status)
- [Roadmap](#roadmap)

---

## Features

- **Multi-agent ticket triage.** A LangGraph workflow with seven nodes: supervisor, diagnosis, RAG, security, resolution, reviewer and ticket.
- **Retrieval-Augmented Generation.** Upload PDF, Word and Excel files, which are chunked, embedded with `bge-m3` and stored in PostgreSQL with **pgvector**. Agents cite the matching chunks.
- **Risk-aware automation.** The security agent rates each case `low`, `medium`, `high` or `critical`, and risky resolutions are flagged as requiring human approval, each with a rollback plan.
- **Self-correcting reviewer.** A reviewer agent can approve a plan or send it back to diagnosis for another pass (bounded retries and a recursion limit prevent loops).
- **Real-time assistant.** A WebSocket endpoint streams workflow progress (`agent_started`, `approval_required`, and so on) to the chat UI.
- **Jira integration.** Create, read, update and comment on Jira issues, and sync ticket status, with every action written to an audit log.
- **Background ingestion.** Heavy document ingestion can be offloaded to **Celery** workers over Redis.
- **Auth and roles.** JWT authentication with bcrypt password hashing and three roles: `employee`, `it_support`, `it_admin`.
- **Observability.** Per-agent run tracking in the database, optional **LangSmith** tracing, structured logging, and `/health` endpoints for the API, database, Redis and Ollama.
- **Modern dashboard.** Tickets, AI assistant, documents, approvals, Jira, monitoring and an admin area, built with Next.js, Tailwind CSS and Recharts.

## How it works

Each ticket flows through a LangGraph state machine:

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

1. **Supervisor** classifies the request (category and priority).
2. **Diagnosis** analyses the likely root cause.
3. **RAG** pulls the most relevant chunks from your uploaded documents (top-k similarity search).
4. **Security** scores the operational risk and decides whether a human must approve.
5. **Resolution** produces a proposed action, ordered steps, the expected result and a rollback plan.
6. **Reviewer** returns `approved`, `rejected` or `request_more_information`. A rejection loops back to diagnosis once.
7. **Ticket** writes the outcome back. The ticket becomes `resolved`, or `in_progress` when approval is required.

All model calls use structured output (Pydantic schemas), and each agent's prompt lives in `backend/src/backend/ai/prompts/`.

## Architecture

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

The backend uses a layered design (`api` → `services` → `repositories` → `db`), so HTTP handling, business logic and persistence stay separate.

## Tech stack

| Layer | Technology |
| --- | --- |
| **Backend** | Python 3.14, FastAPI, Uvicorn, Pydantic v2, SQLAlchemy 2 (async), Alembic |
| **AI / agents** | LangGraph, LangChain, Ollama (`gemma3:4b` for generation, `bge-m3` for embeddings), LangSmith (optional) |
| **Data** | PostgreSQL with pgvector (asyncpg), Redis, Celery |
| **Document parsing** | pypdf, python-docx, openpyxl, BeautifulSoup, tiktoken |
| **Auth** | JWT (python-jose), bcrypt / passlib |
| **Integrations** | Jira (atlassian-python-api) |
| **Frontend** | Next.js 16, React 19, TypeScript, Tailwind CSS 4, Axios, Recharts, lucide-react |
| **Tooling** | uv, pytest, ruff, black, mypy, ESLint |

## Project structure

```
OpsMind/
├── backend/
│   ├── alembic/                 # Database migrations (incl. pgvector setup)
│   ├── src/backend/
│   │   ├── main.py              # FastAPI app entry point
│   │   ├── api/                 # Routes (auth, tickets, documents, jira, workflows, ws, health)
│   │   ├── agents/              # LangGraph workflow, nodes, state, run tracking
│   │   ├── ai/                  # Ollama LLM and embedding clients, schemas, prompt files
│   │   ├── rag/                 # Upload, text extraction, chunking, retrieval
│   │   ├── integrations/jira/   # Jira client and sync service
│   │   ├── services/            # Business logic
│   │   ├── repositories/        # Data access
│   │   ├── db/                  # Engine, session, Redis, ORM models
│   │   ├── worker/              # Celery app and tasks
│   │   ├── schemas/             # Pydantic request/response models
│   │   └── core/                # Config, security, logging, error handling
│   ├── tests/                   # pytest suite
│   ├── pyproject.toml
│   └── requirements.txt
└── frontend/
    ├── app/                     # Next.js routes (dashboard, tickets, assistant, ...)
    ├── components/              # App shell and UI primitives
    └── lib/                     # API client, auth, hooks, types
```

## Getting started

### Prerequisites

- **Python 3.14** and [**uv**](https://docs.astral.sh/uv/) (or `pip`)
- **Node.js 20+** and npm
- **PostgreSQL** with the [pgvector](https://github.com/pgvector/pgvector) extension available
- **Redis** (or Valkey)
- [**Ollama**](https://ollama.com) running locally
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

### 3. Set up the backend

```bash
cd backend

# Install dependencies (uv recommended)
uv sync
# or: python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt && pip install -e .

# Create your environment file (see Configuration below)
cp .env.example .env   # if you don't have an example file, create backend/.env by hand

# Create the database, then apply migrations (this also enables the pgvector extension)
alembic upgrade head

# Run the API
uv run uvicorn backend.main:app --reload --port 8000
```

The interactive API docs are at **http://localhost:8000/docs**.

### 4. (Optional) Start the Celery worker

Only needed if you ingest documents through the background task endpoint.

```bash
cd backend
uv run celery -A backend.worker.celery_app.celery_app worker --loglevel=info
```

### 5. Set up the frontend

```bash
cd frontend
npm install
npm run dev
```

Open **http://localhost:3000**, register an account, and create your first ticket.

> The frontend proxies `/api/*` and `/health/*` to the backend. It defaults to `http://127.0.0.1:8000`. Override it with `NEXT_PUBLIC_API_BASE_URL`.

## Configuration

Settings are loaded from `backend/.env`. Only `DATABASE_URL` is required.

```env
# --- Required ---
DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/opsmind

# --- Security (change in any non-local environment) ---
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

# --- Jira (optional; the integration is skipped when unset) ---
JIRA_BASE_URL=https://your-company.atlassian.net
JIRA_EMAIL=you@company.com
JIRA_API_TOKEN=your-token
JIRA_PROJECT_KEY=IT

# --- LangSmith tracing (optional) ---
LANGSMITH_TRACING=false
LANGSMITH_API_KEY=
LANGSMITH_PROJECT=opsmind
```

> **Never commit `.env`.** It is already ignored by `.gitignore`.

## API overview

| Area | Endpoints |
| --- | --- |
| **Auth** | `POST /api/v1/auth/register` · `POST /api/v1/auth/login` · `GET /api/v1/auth/me` |
| **Users** | `GET /api/v1/users/me` · `GET /api/v1/users/{id}` |
| **Tickets** | `POST/GET /api/v1/tickets` · `GET/PATCH /api/v1/tickets/{id}` · `POST /api/v1/tickets/{id}/messages` |
| **Documents** | `POST /api/v1/documents/upload` · `POST /api/v1/documents/{id}/ingest` · `POST /api/v1/documents/{id}/ingest-task` · `GET /api/v1/documents` · `GET /api/v1/documents/{id}` |
| **Workflows** | `POST /api/v1/workflows/tickets/{ticket_id}/run` |
| **Real-time** | `WS /api/v1/ws/chat?token=<jwt>` |
| **Jira** | `POST /api/v1/jira/issues` · `GET/PATCH /api/v1/jira/issues/{key}` · `POST /api/v1/jira/issues/{key}/comments` · `POST /api/v1/jira/tickets/{id}/sync-status` |
| **Health** | `GET /health` · `/health/db` · `/health/redis` · `/health/ollama` |

Full request and response schemas are available in the Swagger UI at `/docs`.

### WebSocket example

Connect to `ws://localhost:8000/api/v1/ws/chat?token=<jwt>` and send:

```json
{ "ticket_id": "3f1c2b0e-0000-0000-0000-000000000000" }
```

You will receive events such as `connected`, `message_received`, `agent_started`, and `approval_required` (with the workflow ID and risk level).

## Roles and security

| Role | Intended for |
| --- | --- |
| `employee` | Raising and following their own tickets |
| `it_support` | Working tickets, using the Jira integration |
| `it_admin` | Full access, including the admin area |

Jira endpoints require `it_support` or `it_admin`.

- Passwords are hashed with **bcrypt**, and sessions use short-lived **JWTs** (60 minutes by default).
- Jira actions and sync failures are recorded in an **audit log** table.
- Agents are prompted never to recommend unsafe commands, and anything the security agent flags is routed to a human.
- **Before deploying:** set a strong `JWT_SECRET_KEY` (the default is a placeholder), restrict `CORS_ORIGINS`, and serve the app over HTTPS.

## Testing and code quality

```bash
cd backend

uv run pytest          # run the test suite (auth, models, API routes, health)
uv run ruff check .    # lint
uv run black .         # format
uv run mypy src        # type-check

cd ../frontend
npm run lint
```

## Project status

OpsMind is under active development. The backend platform is largely complete, and parts of the dashboard are still wired up to placeholder data.

| Area | Status |
| --- | --- |
| Auth, users and roles | Done |
| Tickets and messages | Done |
| Document upload, ingestion and RAG retrieval | Done |
| LangGraph multi-agent workflow | Done |
| WebSocket assistant | Done |
| Jira integration (API) | Done |
| Health checks (API, DB, Redis, Ollama) | Done |
| Approval queue UI | UI built, uses sample data; approve/reject API pending |
| Jira health and agent-failure dashboard | Pending backend endpoints |

## Roadmap

- Backend approval API (approve and reject) wired to the Approvals page
- Agent failure and run history endpoints for the Monitoring page
- Jira health check
- Pluggable monitoring connectors (Datadog, Grafana and others) for automated incident context
- Role-based ticket visibility and an admin user-management UI
- Docker Compose setup for one-command local startup
- CI pipeline (tests, lint, type-check)

---

Built by [Tiwari-Tech](https://github.com/Tiwari-Tech). Contributions, issues and ideas are welcome.
