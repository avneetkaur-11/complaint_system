# Complaints AI Teammate — Multi-Agent Client Complaint Management System

An AI-orchestrated client complaint (case) management system. The backend is built with **Python, FastAPI, and SQLAlchemy**, backed by **PostgreSQL + pgvector** for semantic retrieval, exposed through both **REST** and **GraphQL** APIs, and driven by a **multi-agent AI Orchestrator** that classifies, investigates, and recommends resolutions for each case using a **RAG (Retrieval-Augmented Generation)** pipeline.

> **Note:** The AI/LLM calls in this project are currently **mocked**. The multi-agent orchestrator, embeddings, and recommendation generation are stubbed with deterministic mock logic so the full end-to-end flow can be demoed and tested without a live LLM/API key. Swapping in a real model (Qwen, Gemma, GPT, Claude, etc.) only requires replacing the `classification_service` / `retrieval_service` implementations — the surrounding architecture stays the same.

---

## Tech Stack

| Layer            | Technology                                        |
|-------------------|----------------------------------------------------|
| Backend framework | FastAPI                                            |
| ORM               | SQLAlchemy 2.0                                     |
| Database          | PostgreSQL                                         |
| Vector store      | pgvector extension (semantic case/knowledge search)|
| API layer         | REST + GraphQL (single `/api/graphql` endpoint)    |
| AI Orchestrator   | Multi-agent pipeline (currently mocked)            |
| RAG               | Mocked embeddings + pgvector similarity retrieval  |
| Frontend          | React (Customer Portal / Agent Portal / Admin Portal) |
| Auth              | OIDC / JWT                                          |
| Server            | Uvicorn (ASGI)                                     |

---

## Project Structure

```
backend/
└── app/
    ├── main.py                     # FastAPI app entrypoint
    ├── config.py                   # Settings (env vars)
    ├── db.py                       # SQLAlchemy engine/session
    ├── models/
    │   ├── case_model.py           # Case (complaint) ORM model
    │   ├── agent_finding_model.py  # Findings produced by each AI agent
    │   ├── approval_item_model.py  # Human-in-the-loop approval queue
    │   └── knowledge_base_model.py # KB documents + embeddings
    ├── routers/
    │   ├── complaints.py           # Case intake, update, retrieval
    │   ├── approval_queue.py       # Human approval decisions
    │   ├── knowledge_base.py       # KB document management
    │   └── graphql_api.py          # GraphQL endpoint
    ├── services/
    │   ├── case_service.py         # Case lifecycle / business logic
    │   ├── classification_service.py # AI Orchestrator: classify/investigate/recommend
    │   ├── retrieval_service.py    # RAG retrieval (pgvector similarity search)
    │   └── email_service.py        # Customer/agent communication
    └── schemas.py                  # Pydantic request/response models
frontend/                            # React (Customer / Agent / Admin portals)
```

---

## 1. High-Level Solution Architecture

Customers interact through **Web, Mobile, or Email/RM channels**, which route into three React portals — a **Customer Portal**, an **Agent Portal**, and an **Admin Portal**. All portals go through authentication (OIDC/JWT) and an API Gateway layer (routing, rate limiting, auth, monitoring, logging) before reaching the **Complaint Management API**.

The backend is organized into three cooperating services:

- **Complaint Service** — create/update, categorization, prioritization, intake
- **Case Service** — case management, timeline, notes & history, escalation
- **Communication Service** — email/SMS/WhatsApp notifications, template management

These feed into the **AI Orchestrator Service**, which coordinates the multi-agent framework (below) against a PostgreSQL database and external enterprise systems (Core Banking, CRM, Payment System, Fraud Engine, KYC, Document Repo) as well as external communication channels (Email, SMS, WhatsApp).

```mermaid
flowchart TB
    Customers["Customers (Web / Mobile / Email / RM)"] --> Portals

    subgraph Portals["Frontend Portals (React)"]
        CP[Customer Portal]
        AP[Agent Portal]
        ADP[Admin Portal]
    end

    Portals --> Auth["Authentication (OIDC / JWT)"]
    Auth --> GW["API Gateway (Routing, Rate Limiting, Auth, Monitoring, Logging)"]

    GW --> ComplaintAPI["Complaint Management API"]

    subgraph ComplaintAPI["Complaint Management API"]
        CS["Complaint Service<br/>Create/Update, Categorization, Prioritization, Intake"]
        CaseS["Case Service<br/>Case Mgmt, Timeline, Notes & History, Escalation"]
        Comm["Communication Service<br/>Email/SMS/WhatsApp, Notification Engine, Templates"]
    end

    ComplaintAPI --> Orchestrator["AI Orchestrator Service"]

    subgraph MultiAgent["Multi-Agent Framework"]
        Classifier[Classifier Agent]
        Investigation[Investigation Agent]
        Evidence[Evidence Agent]
        RootCause[Root Cause Agent]
        Recommendation[Recommendation Agent]
    end

    Orchestrator --> MultiAgent
    MultiAgent <--> KB["Knowledge Retrieval (RAG Engine)"]

    MultiAgent --> PG[("PostgreSQL Database<br/>Complaint/Case/Customer/Timeline/Notes/Communications/Documents/Audit Logs/Agent Decisions/Approvals/Users/Roles")]
    ComplaintAPI --> Enterprise["Enterprise Systems (APIs)<br/>Core Banking, CRM, Payment System, Fraud Engine, KYC, Document Repo"]
    Comm --> External["External Channels<br/>Email Server, SMS Gateway, WhatsApp API"]
```

---

## 2. AI Orchestrator & Multi-Agent Architecture

Each incoming complaint flows into the **AI Orchestrator**, which receives the complaint, creates an investigation workflow, coordinates agents, tracks progress, and produces a final decision. Input sources include the customer complaint itself (web/mobile), email, and RM/chat channels.

The orchestrator runs a chain of specialized agents:

1. **Complaint Understanding Agent** — extracts intent, detects category, detects urgency, extracts entities
2. **Investigation Agent** — queries enterprise APIs, collects transactions/history, finds similar complaints
3. **Evidence Agent** — collects documents, screenshots, logs, database records
4. **Root Cause Agent** — finds the failure point, policy violation, missing document, or technical issue
5. **Recommendation Agent** — proposes resolution options: refund/adjustment, escalation, or compensation
6. **Human Approval Agent** — routes to manager/operations approval and compliance check
7. **Communication Agent** — sends email/SMS/WhatsApp updates to the customer using templates

Agents 1–5 are backed by a **Knowledge Retrieval (RAG) engine**: policy documents, SOPs, previous complaints, resolution history, and knowledge articles are embedded and stored in a **pgvector**-backed vector database; each agent's queries retrieve relevant context before reasoning, which is then passed to an **LLM / AI Factory** (Qwen / Gemma / GPT / Claude — model selected per use case; currently mocked).

```mermaid
flowchart LR
    subgraph Inputs["Input Sources"]
        IC[Customer Complaint]
        IWM[Web/Mobile]
        IEM[Email]
        IRM[RM / Chat]
    end

    Inputs --> Orchestrator["AI Orchestrator<br/>Receives complaint, builds workflow,<br/>coordinates agents, tracks progress,<br/>makes final decision"]

    Orchestrator --> A1[1. Complaint Understanding Agent]
    A1 --> A2[2. Investigation Agent]
    A2 --> A3[3. Evidence Agent]
    A3 --> A4[4. Root Cause Agent]
    A4 --> A5[5. Recommendation Agent]

    subgraph RAG["Knowledge Retrieval (RAG)"]
        Docs["Policy Docs / SOPs / Prior Complaints / Resolution History / KB Articles"]
        Embed[Embedding Model]
        VDB[("Vector Database (pgvector)")]
        Ctx[Retrieved Context]
        Docs --> Embed --> VDB --> Ctx
    end

    A1 <--> RAG
    A2 <--> RAG
    A3 <--> RAG
    A4 <--> RAG
    A5 <--> RAG

    A5 --> A6[6. Human Approval Agent]
    A6 --> A7[7. Communication Agent]

    RAG -.-> LLM["LLM / AI Factory<br/>Qwen / Gemma / GPT / Claude<br/>(mocked)"]
```

---

## 3. End-to-End Flow (Complaint Lifecycle)

```mermaid
flowchart LR
    S1((Customer Submits Complaint)) --> S2((Complaint Service))
    S2 --> S3((Create Case))
    S3 --> S4((Store in PostgreSQL))
    S4 --> S5((Trigger AI Orchestrator))
    S5 --> O1[Multi-Agent Pipeline<br/>Classify → Investigate → RAG → Evidence → Recommend]
    O1 --> S6((Human Approval))
    S6 --> S7((Customer Communication))
    S7 --> S8((Case Closed & Audit Stored))
```

---

## Frontend

The frontend is built with **React**, split into three portals sharing the same backend:

- **Customer Portal** — submit complaints, track case status, view resolution
- **Agent Portal** — work the case queue, review AI findings/recommendations, add notes
- **Admin Portal** — manage knowledge base documents, review approvals, view audit logs

The frontend talks to the backend primarily via **GraphQL** (`/api/graphql`) for queries and mutations, with REST endpoints available for simpler operations.

In development, proxy `/api` to `http://localhost:8000` via your Vite config:

```ts
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      "/api": "http://localhost:8000",
    },
  },
});
```

### Run the frontend

```bash
cd frontend
npm install
npm run dev
```

---

## AI Orchestrator / RAG Layer (Mocked)

The AI layer simulates the full multi-agent + RAG pipeline described above so the architecture and data flow are production-representative, without requiring API keys or GPU inference:

1. **Embedding generation** — mock embedding vectors are generated for complaint text and knowledge base documents, stored via **pgvector** columns.
2. **Retrieval** (`retrieval_service.py`) — uses pgvector similarity search to find relevant knowledge base articles and similar past cases for the current complaint.
3. **Classification & Recommendation** (`classification_service.py`) — mocks the Classifier, Investigation, Root Cause, and Recommendation agents, returning a deterministic category, sub-category, severity, summary, and a recommended action with a confidence score.

To plug in a real model later: swap the mock logic in `classification_service.py` / `retrieval_service.py` for real embedding + LLM calls (Qwen, Gemma, GPT, Claude, etc.) — the surrounding orchestration, storage, and approval flow stays unchanged.

---

## Email Notifications (SMTP)

`email_service.py` sends the customer a **notification email over SMTP** every time a case's status changes — intake acknowledgment, moving through investigation, recommendation approved/rejected, and final resolution/closure. This gives the customer a real-time trail of updates without needing to check the portal.

---

## Getting Started — Run Locally

### 1. Backend setup

```bash
# 1. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate      # on Windows: venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Start PostgreSQL with the pgvector extension enabled
docker run -d --name complaints-db -p 5432:5432 \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=complaints \
  ankane/pgvector

# 4. Set the database URL
export DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/complaints
# on Windows (cmd): set DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/complaints

# 5. Enable the pgvector extension inside the DB (one-time)
psql $DATABASE_URL -c "CREATE EXTENSION IF NOT EXISTS vector;"

# 6. Start the API
uvicorn app.main:app --reload --port 8000
```

The API will be available at `http://localhost:8000`.
Interactive REST docs (Swagger UI): `http://localhost:8000/docs`
GraphQL endpoint: `http://localhost:8000/api/graphql`

### 2. Frontend setup

```bash
cd frontend
npm install
npm run dev
```

Frontend runs at `http://localhost:5173` (default Vite port) and proxies API calls to the backend.

### Environment Variables

| Variable        | Description                                  | Example                                                              |
|------------------|-----------------------------------------------|-----------------------------------------------------------------------|
| `DATABASE_URL`   | Postgres connection string                    | `postgresql+psycopg://postgres:postgres@localhost:5432/complaints`   |
| `AI_MODE`        | `mock` (default) or `live`                    | `mock`                                                                |
| `EMBEDDING_DIM`  | Dimension of mock embedding vectors           | `384`                                                                |
| `SMTP_HOST`      | SMTP server host for status-update emails     | `smtp.gmail.com`                                                      |
| `SMTP_PORT`      | SMTP server port                              | `587`                                                                 |
| `SMTP_USERNAME`  | SMTP account username                         | `notifications@example.com`                                           |
| `SMTP_PASSWORD`  | SMTP account password / app password          | `********`                                                            |
| `SMTP_FROM`      | "From" address used on outgoing emails        | `noreply@complaints.example.com`                                      |

---

## API Endpoints

### REST Endpoints

#### Complaints / Cases (`routers/complaints.py`)

| Method | Endpoint                                | Description                                                   |
|--------|-------------------------------------------|------------------------------------------------------------------|
| POST   | `/api/complaints`                         | Submit a new complaint (`ComplaintIntakeRequest`) → creates a Case, triggers AI Orchestrator |
| GET    | `/api/complaints`                         | List all cases                                                  |
| GET    | `/api/complaints/{caseId}`                | Get a single case (`CaseResponse`) with classification/recommendation |
| PUT    | `/api/complaints/{caseId}`                | Update case details (`ComplaintUpdateRequest`)                  |
| GET    | `/api/complaints/{caseId}/findings`       | Get all AI agent findings for a case (`AgentFindingResponse[]`) |

#### Approval Queue (`routers/approval_queue.py`)

| Method | Endpoint                                   | Description                                             |
|--------|-----------------------------------------------|-------------------------------------------------------------|
| GET    | `/api/approvals`                              | List pending approval queue items (`ApprovalQueueItem[]`)   |
| GET    | `/api/approvals/{approvalId}`                 | Get a single approval item with its case details            |
| POST   | `/api/approvals/{approvalId}/approve`         | Approve a recommendation (`ApprovalDecisionRequest`)         |
| POST   | `/api/approvals/{approvalId}/reject`          | Reject a recommendation (`ApprovalDecisionRequest`)          |

#### Knowledge Base (`routers/knowledge_base.py`)

| Method | Endpoint                              | Description                                                    |
|--------|------------------------------------------|----------------------------------------------------------------|
| POST   | `/api/knowledge-base`                    | Upload/create a KB document (`KnowledgeBaseCreateRequest`) — chunked and embedded into pgvector |
| GET    | `/api/knowledge-base`                    | List all KB documents (`KnowledgeBaseResponse[]`)               |
| GET    | `/api/knowledge-base/{docId}`            | Get a single KB document's details                              |
| DELETE | `/api/knowledge-base/{docId}`            | Delete a KB document and its indexed chunks                     |

#### Health

| Method | Endpoint  | Description       |
|--------|-----------|--------------------|
| GET    | `/health` | Health check       |

### GraphQL Endpoint (`routers/graphql_api.py`)

Single endpoint: `POST /api/graphql` (accepts `GraphQLRequest { query, variables }`) — supports queries and mutations for cases, agent findings, and the approval queue.

---

## Database Schema (Core Tables)

- **cases** — `caseId, clientId, channel, rawDescription, category, subCategory, severity, summary, status, recommendedAction, recommendationConfidence, clientName, clientEmail, clientPhone, createdAt, updatedAt`
- **agent_findings** — `findingId, caseId, agentName, findingType, content, confidence, retrievedContextIds, createdAt`
- **approval_items** — `approvalId, caseId, status, decidedBy, decidedAt`
- **knowledge_base_documents** — `docId, title, docType, content, chunksIndexed, embedding vector(384)` (pgvector column)
