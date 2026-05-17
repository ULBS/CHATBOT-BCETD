# BCETD Chatbot — Final Project Structure

This is the **assembled, run-tested project** after all 5 members have integrated their work.

```
chatbot-bcetd/
│
│  ── ROOT FILES (Member 1 — Infrastructure) ──
├── docker-compose.yml          Defines all 4 Docker services
├── .env.example                Template for credentials
├── .gitignore                  Files Git should ignore
├── README.md                   Project overview
│
│  ── MEMBER 1: DevOps & Infrastructure ──
├── scripts/
│   ├── start.sh                One-command startup with health checks
│   └── healthcheck.sh          Service health monitor
├── member1-infrastructure/
│   └── grafana/
│       └── datasources.yml     Grafana → PostgreSQL auto-config
├── .github/workflows/
│   └── ci.yml                  GitHub Actions CI pipeline
│
│  ── MEMBER 2: RAG Pipeline Engineer ──
├── member2-rag-pipeline/
│   ├── workflow_ingestion_pipeline.json    Documents → vectors → Qdrant
│   ├── workflow_query_pipeline.json        Webhook → guardrails → search → LLM → response
│   └── SYSTEM_PROMPT_DOCUMENTATION.md      LLM prompt explanation
│
│  ── MEMBER 3: Ethics & Guardrails Engineer ──
├── member3-ethics-guardrails/
│   ├── workflow_guardrails.json            Sub-workflow: Layer 1 + Layer 2
│   ├── workflow_output_validation.json     Sub-workflow: Layer 3
│   ├── blocklist.txt                       Inappropriate terms
│   ├── adversarial_test_suite.json         57 adversarial test cases
│   ├── test_guardrails.sh                  Automated test runner
│   ├── fallback_messages.md                Rejection message templates
│   └── GUARDRAILS_DOCUMENTATION.md         3-layer system specification
│
│  ── MEMBER 4: Analytics & Database Engineer ──
├── member4-analytics/
│   ├── 001_initial_schema.sql              PostgreSQL tables, views, functions
│   ├── workflow_analytics_logging.json     Sub-workflow: async logging
│   ├── workflow_daily_stats.json           Cron 00:05: aggregate + purge
│   ├── workflow_analytics_api.json         5 webhook endpoints for dashboard
│   ├── grafana_dashboard.json              9-panel pre-built dashboard
│   ├── backup_database.sh                  Daily DB dump with rotation
│   └── PRIVACY_DOCUMENTATION.md            GDPR compliance notes
│
│  ── MEMBER 5: Frontend & Integration Engineer ──
├── member5-frontend-python/                Python Flask web application
│   ├── Dockerfile                          Container build (gunicorn)
│   ├── requirements.txt                    pip dependencies
│   ├── setup.cfg                           pytest config
│   ├── run.py                              Entry point
│   ├── app/
│   │   ├── __init__.py                     Flask application factory
│   │   ├── config.py                       Env-based configuration
│   │   ├── routes/
│   │   │   ├── __init__.py
│   │   │   ├── chat.py                     GET / → chat page
│   │   │   ├── admin.py                    GET /admin/ → dashboard
│   │   │   └── api.py                      POST /api/chat, GET /api/analytics/*
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   ├── n8n_client.py               HTTP client to n8n
│   │   │   └── session_manager.py          Rotating session IDs
│   │   ├── templates/                      Jinja2 templates
│   │   │   ├── base.html
│   │   │   ├── chat.html
│   │   │   └── admin.html
│   │   └── static/
│   │       ├── css/chat.css                ULBS-branded styling
│   │       └── js/
│   │           ├── chat.js                 Chat input + message rendering
│   │           └── admin.js                Dashboard charts
│   └── tests/
│       ├── __init__.py
│       └── test_app.py                     30 unit tests
│
│  ── SHARED RESOURCES ──
├── shared/
│   └── api_contract.json                   Webhook API specification
├── docs/
│   ├── INSTALLATION_TUTORIAL.md            Detailed install guide
│   ├── BRANCHING_STRATEGY.md               Git workflow
│   └── DOCUMENT_CORPUS_GUIDE.md            How to prepare documents
│
│  ── RUNTIME DATA (created during operation) ──
├── data/
│   └── documents/              Place ULBS PDFs/DOCX/TXT here
└── backups/
    └── postgres/               Daily database backups land here
```

---

## What runs at runtime

The `docker-compose.yml` spins up these 4 containers:

| Container | Image | Port | Purpose |
|---|---|---|---|
| `bcetd-frontend` | Built from `member5-frontend-python/Dockerfile` | 3000 | Python Flask UI |
| `bcetd-n8n` | `n8nio/n8n:latest` | 5678 | Workflow engine |
| `bcetd-qdrant` | `qdrant/qdrant:latest` | 6333 | Vector database |
| `bcetd-postgres` | `postgres:16-alpine` | 5432 | Analytics database |

Optional containers (started with `--profile`):
- `bcetd-ollama` (port 11434) — local LLM alternative to OpenAI
- `bcetd-grafana` (port 3001) — analytics dashboards

---

## Final file count by member

| Member | Files | Role |
|---|---|---|
| Member 1 (DevOps) | 9 | Infrastructure, scripts, CI |
| Member 2 (RAG) | 3 | Ingestion + query workflows |
| Member 3 (Ethics) | 7 | Guardrails, tests, blocklist |
| Member 4 (Analytics) | 7 | Schema, workflows, dashboards |
| Member 5 (Frontend) | 21 | Flask app, templates, tests |
| Shared/docs | 5 | API contract, tutorials |
| **Total** | **52** | |

---

## What was tested and works

All of these have been verified end-to-end:

- ✅ `docker compose up -d --build` brings up all 4 core services
- ✅ PostgreSQL auto-runs `001_initial_schema.sql` on first boot (creates 4 tables, 6 views, 3 functions)
- ✅ All 7 n8n workflows import cleanly
- ✅ Sub-workflow ID wiring (M3 guardrails + M4 analytics → M2 query pipeline) works
- ✅ Document ingestion pipeline reads from `data/documents/`, embeds via OpenAI, stores in Qdrant
- ✅ Query pipeline returns answers grounded in documents with source citations
- ✅ Layer 1 regex catches prompt injection (e.g. "Ignore all previous instructions")
- ✅ Layer 2 LLM classification rejects off-topic queries (e.g. "best chocolate cake recipe")
- ✅ Layer 3 output validation catches hallucinations
- ✅ Analytics logging stores SHA-256 hashed queries (zero PII)
- ✅ Daily stats cron runs at 00:05 (aggregates + purges >90 days)
- ✅ Admin dashboard at `/admin/` displays 4 KPIs + 3 charts + 2 tables
- ✅ Chat UI displays bilingual welcome, suggestion chips, source citations, confidence

---

## Quick start (after you have this folder)

```powershell
cd $HOME\Desktop\chatbot-bcetd
Copy-Item .env.example .env
notepad .env                         # add OPENAI_API_KEY
docker compose up -d --build
```

Then open http://localhost:5678 to configure n8n (see `SETUP-GUIDE.md` outside this folder for the click-by-click instructions).

After n8n is configured, the chatbot is at http://localhost:3000 and the admin dashboard at http://localhost:3000/admin.
