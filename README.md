# CHATBOT BCETD Agent

**AI-Powered University Document Assistant for Universitatea Lucian Blaga din Sibiu**

An intelligent chatbot that answers student questions exclusively from official university documents using RAG (Retrieval-Augmented Generation). It never hallucates — if the documents don't contain the answer, it says so.

---

## Quick Start (5 Minutes)

### Prerequisites

- [Docker Desktop](https://docs.docker.com/get-docker/) (v24+) — or Docker Engine + Docker Compose V2 on Linux
- [Git](https://git-scm.com/downloads)
- An OpenAI API key ([get one here](https://platform.openai.com/api-keys)) — or use Ollama for fully local operation

### Setup

```bash
# 1. Clone the repository
git clone https://github.com/your-org/chatbot-bcetd.git
cd chatbot-bcetd

# 2. Configure environment
cp .env.example .env
# Edit .env and add your OPENAI_API_KEY

# 3. Start everything
chmod +x scripts/start.sh
./scripts/start.sh
```

That's it. The startup script handles building containers, checking health, and verifying ports.

### Access Points

| Service | URL | Purpose |
|---------|-----|---------|
| **Chat UI** | http://localhost:3000 | Student-facing chatbot |
| **Admin Dashboard** | http://localhost:3000/admin | Analytics & metrics |
| **n8n Workflows** | http://localhost:5678 | AI pipeline editor |
| **Qdrant Dashboard** | http://localhost:6333/dashboard | Vector database inspector |
| **Grafana** | http://localhost:3001 | Advanced analytics (optional) |

---

## Architecture

```
  Student                Admin
    │                      │
    ▼                      ▼
┌──────────┐       ┌──────────────┐
│ Chat UI  │       │ Admin Panel  │
│ :3000    │       │ :3000/admin  │
└────┬─────┘       └──────┬───────┘
     │ POST /webhook/chat  │ GET /analytics/*
     ▼                     ▼
┌──────────────────────────────────────────┐
│              n8n  :5678                  │
│  ┌───────────┐  ┌──────────┐            │
│  │ Guardrails│  │Query     │            │
│  │ (M3)      │──│Pipeline  │            │
│  │ L1+L2+L3  │  │(M2)     │            │
│  └───────────┘  └─────┬────┘            │
│                       │                  │
│            ┌──────────┼──────────┐       │
│            ▼          ▼          ▼       │
│     ┌──────────┐ ┌────────┐ ┌────────┐  │
│     │ Qdrant   │ │  LLM   │ │Postgres│  │
│     │ :6333    │ │OpenAI / │ │:5432   │  │
│     │ vectors  │ │Ollama   │ │analytics│ │
│     └──────────┘ └────────┘ └────────┘  │
└──────────────────────────────────────────┘
```

---

## Setup Guide (Detailed)

### Step 1: Environment Configuration

```bash
cp .env.example .env
```

**Required:** Set your `OPENAI_API_KEY` in `.env`. All other values have working defaults.

**Change in production:** All passwords (`N8N_PASSWORD`, `POSTGRES_PASSWORD`, `GRAFANA_PASSWORD`).

### Step 2: Launch Services

```bash
# Core only (recommended for getting started)
./scripts/start.sh

# Core + local LLM (no OpenAI needed)
./scripts/start.sh --ollama

# Core + Grafana monitoring
./scripts/start.sh --monitoring

# Everything
./scripts/start.sh --all
```

### Step 3: Import n8n Workflows

1. Open **http://localhost:5678** (login: admin / changeme_n8n)
2. Go to **Workflows → Import from File**
3. Import in this order:

| Order | File | Notes |
|-------|------|-------|
| 1 | `member3-ethics-guardrails/workflow_guardrails.json` | Note the workflow ID |
| 2 | `member3-ethics-guardrails/workflow_output_validation.json` | Note the workflow ID |
| 3 | `member4-analytics/workflow_analytics_logging.json` | Note the workflow ID |
| 4 | `member2-rag-pipeline/workflow_ingestion_pipeline.json` | |
| 5 | `member2-rag-pipeline/workflow_query_pipeline.json` | Update sub-workflow IDs |
| 6 | `member4-analytics/workflow_daily_stats.json` | Activate (scheduled) |
| 7 | `member4-analytics/workflow_analytics_api.json` | Activate |

4. In the **query pipeline** workflow, update placeholder IDs:
   - `GUARDRAILS_WORKFLOW_ID` → ID from step 1
   - `OUTPUT_VALIDATION_WORKFLOW_ID` → ID from step 2

5. Add credentials in n8n → **Credentials**:
   - **OpenAI API**: paste your API key
   - **PostgreSQL**: host=`postgres`, port=`5432`, db=`chatbot_stats`, user=`chatbot_user`, password from `.env`

### Step 4: Add University Documents

Place PDF, DOCX, TXT, or HTML files in the `data/documents/` folder:

```
data/documents/
├── calendarul_academic_2025_2026.pdf
├── regulament_studii_licenta.pdf
├── taxe_scolarizare.pdf
└── ...
```

See `member5-frontend/DOCUMENT_CORPUS_GUIDE.md` for detailed preparation instructions.

### Step 5: Run Document Ingestion

1. Open the **Document Ingestion Pipeline** workflow in n8n
2. Click **Execute Workflow**
3. Verify: `curl http://localhost:6333/collections/ulbs_documents` should show the chunk count

### Step 6: Test the Chatbot

Open **http://localhost:3000** and ask a question!

---

## Service Management

```bash
# Check status of all services
./scripts/start.sh --status

# Stop all services (data preserved)
./scripts/start.sh --stop

# Restart
./scripts/start.sh

# Full reset — DELETE ALL DATA (requires typing RESET)
./scripts/start.sh --reset

# View logs for a specific service
docker logs bcetd-n8n --tail 50 -f
docker logs bcetd-qdrant --tail 50 -f
docker logs bcetd-postgres --tail 50 -f
```

---

## Using Ollama (Local LLM — No API Key Needed)

For fully offline operation with zero data leaving your machine:

```bash
# Start with Ollama profile
./scripts/start.sh --ollama

# Pull required models (first time only, ~4-8 GB downloads)
docker exec bcetd-ollama ollama pull llama3.1
docker exec bcetd-ollama ollama pull nomic-embed-text

# Verify models are available
docker exec bcetd-ollama ollama list
```

Then in the n8n workflows:
1. Disable the "OpenAI Embeddings" node, enable "Ollama Embeddings"
2. Replace the "OpenAI GPT-4o-mini" node with an Ollama Chat node pointing to `http://ollama:11434`
3. Change Qdrant vector size from 1536 to 384
4. **Re-run the ingestion pipeline** (vectors from different models are incompatible)

---

## Project Structure

```
chatbot-bcetd/
│
├── docker-compose.yml          ← Infrastructure definition (THIS IS THE KEY FILE)
├── .env.example                ← Environment template (copy to .env)
├── .env                        ← Your secrets (NEVER committed to Git)
├── .gitignore                  ← Git exclusion rules
│
├── scripts/
│   └── start.sh                ← One-command startup with health checks
│
├── data/
│   └── documents/              ← Place ULBS documents here for ingestion
│       └── .gitkeep
│
├── backups/
│   └── postgres/               ← Database backups (auto-generated)
│       └── .gitkeep
│
├── member1-infrastructure/     ← Member 1: DevOps configs
│   └── grafana/
│       └── datasources.yml     ← Auto-provisioned Grafana datasource
│
├── member2-rag-pipeline/       ← Member 2: RAG pipeline workflows
│   ├── workflow_ingestion_pipeline.json
│   ├── workflow_query_pipeline.json
│   └── SYSTEM_PROMPT_DOCUMENTATION.md
│
├── member3-ethics-guardrails/  ← Member 3: Safety & guardrails
│   ├── workflow_guardrails.json
│   ├── workflow_output_validation.json
│   ├── blocklist.txt
│   ├── adversarial_test_suite.json
│   ├── test_guardrails.sh
│   ├── fallback_messages.md
│   └── GUARDRAILS_DOCUMENTATION.md
│
├── member4-analytics/          ← Member 4: Analytics & database
│   ├── 001_initial_schema.sql
│   ├── workflow_analytics_logging.json
│   ├── workflow_daily_stats.json
│   ├── workflow_analytics_api.json
│   ├── grafana_dashboard.json
│   ├── backup_database.sh
│   └── PRIVACY_DOCUMENTATION.md
│
├── member5-frontend/           ← Member 5: UI & integration
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── chat-ui/index.html
│   ├── admin-dashboard/admin.html
│   ├── tests/test_e2e.sh
│   └── DOCUMENT_CORPUS_GUIDE.md
│
├── shared/
│   └── api_contract.json       ← Webhook API specification
│
├── docs/                       ← Additional documentation
│   └── BRANCHING_STRATEGY.md
│
└── .github/
    └── workflows/
        └── ci.yml              ← GitHub Actions CI pipeline
```

---

## Networking

All services communicate over a shared Docker bridge network (`bcetd-network`). Inside the network, services reference each other by container name — no IP addresses needed:

| From | To | Internal URL |
|------|----|-------------|
| n8n | Qdrant | `http://qdrant:6333` |
| n8n | PostgreSQL | `postgres:5432` |
| n8n | Ollama | `http://ollama:11434` |
| n8n | OpenAI | `https://api.openai.com` (external) |
| Frontend | n8n | `http://n8n:5678` (internal) or `http://localhost:5678` (browser) |
| Grafana | PostgreSQL | `postgres:5432` |

**External access** (from your browser) uses `localhost` with the mapped ports.

---

## Data Persistence

All service data is stored in named Docker volumes:

| Volume | Service | Contains |
|--------|---------|----------|
| `bcetd_n8n_data` | n8n | Workflows, credentials, execution history |
| `bcetd_qdrant_data` | Qdrant | Vector embeddings, collection data |
| `bcetd_postgres_data` | PostgreSQL | Analytics tables, query logs |
| `bcetd_ollama_data` | Ollama | Downloaded LLM model files |
| `bcetd_grafana_data` | Grafana | Dashboards, user preferences |

Data survives `docker compose down` and `docker compose up`. To delete all data: `docker compose down -v`.

---

## Testing

```bash
# Guardrail adversarial tests (57 test cases)
chmod +x member3-ethics-guardrails/test_guardrails.sh
./member3-ethics-guardrails/test_guardrails.sh

# End-to-end integration tests
chmod +x member5-frontend/tests/test_e2e.sh
./member5-frontend/tests/test_e2e.sh

# Verify analytics are logging
docker exec bcetd-postgres psql -U chatbot_user -d chatbot_stats \
  -c "SELECT COUNT(*) as total, COUNT(*) FILTER (WHERE was_answered) as answered FROM query_logs;"

# Verify Qdrant has documents
curl -s http://localhost:6333/collections/ulbs_documents | python3 -m json.tool
```

---

## Troubleshooting

### "Cannot connect to server" in Chat UI
The n8n webhook isn't reachable. Check:
```bash
docker logs bcetd-n8n --tail 20
curl http://localhost:5678/healthz
```

### "No relevant documents found" for every question
The Qdrant collection may be empty. Run the ingestion pipeline:
1. Place documents in `data/documents/`
2. Execute the ingestion workflow in n8n

### Port already in use
Edit `.env` and change the conflicting port number (e.g., `N8N_PORT=5679`).

### Ollama model download is slow
Models are 4-8 GB. First download takes time. Progress:
```bash
docker logs bcetd-ollama --tail 5 -f
```

### PostgreSQL won't start after schema change
If you modified the schema SQL after first boot, the init script won't re-run (PostgreSQL only runs `initdb.d` on empty data directories). Reset:
```bash
docker compose down
docker volume rm bcetd_postgres_data
docker compose up -d postgres
```

### n8n shows "Workflow could not be activated"
Usually a missing credential. Go to n8n → Credentials and ensure OpenAI API and PostgreSQL credentials are configured.

---

## Team Responsibilities

| Member | Role | Key Deliverable |
|--------|------|----------------|
| **M1** | Infrastructure & DevOps | `docker-compose.yml`, startup scripts, Git repo |
| **M2** | RAG Pipeline Engineer | n8n ingestion + query workflows, system prompt |
| **M3** | Ethics & Guardrails | 3-layer safety system, adversarial test suite |
| **M4** | Analytics & Database | PostgreSQL schema, Grafana dashboards, privacy docs |
| **M5** | Frontend & Integration | Chat UI, admin dashboard, document corpus |

---

## License

Internal use only — ULBS Engineering Team.
