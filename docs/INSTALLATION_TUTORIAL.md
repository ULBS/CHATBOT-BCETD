# CHATBOT BCETD — Complete Installation & Setup Tutorial

## What This Project Actually Is (Tech Stack Explained)

This is **NOT** a Python project. Here is exactly what each part of the system is built with:

| Component | Technology | Language | Runs Where |
|-----------|-----------|----------|------------|
| **AI Pipeline (RAG, guardrails, prompts)** | n8n (visual workflow tool) | JavaScript (inside n8n code nodes) | Docker container |
| **Vector Database** | Qdrant | Rust (pre-built binary) | Docker container |
| **Analytics Database** | PostgreSQL 16 | SQL | Docker container |
| **Student Chat UI** | Plain HTML + CSS + JS | JavaScript (vanilla, no framework) | Docker container (nginx) |
| **Admin Dashboard** | Plain HTML + Chart.js | JavaScript | Docker container (nginx) |
| **Local LLM (optional)** | Ollama | Go (pre-built binary) | Docker container |
| **Monitoring (optional)** | Grafana | Go (pre-built binary) | Docker container |
| **LLM for answers** | OpenAI GPT-4o-mini | API call (no local code) | OpenAI cloud |
| **Embeddings** | OpenAI text-embedding-3-small | API call (no local code) | OpenAI cloud |
| **Startup scripts** | Bash | Shell | Your host machine |
| **CI Pipeline** | GitHub Actions | YAML | GitHub cloud |

**The key insight:** You don't install Python, Node.js, or any runtime on your machine. Docker runs everything inside isolated containers. The only things you install on your machine are Docker and Git.

---

## What You Need to Install

### 1. Docker Desktop (REQUIRED)

Docker is the **only** mandatory software. It runs all 6 services inside containers.

#### Windows 10/11
1. Download Docker Desktop: https://www.docker.com/products/docker-desktop/
2. Run the installer
3. During setup, enable **WSL 2 backend** (recommended) or Hyper-V
4. Restart your computer when prompted
5. Open Docker Desktop — wait for it to say "Docker is running"
6. Open PowerShell or CMD and verify:
   ```
   docker --version
   docker compose version
   ```

**Windows requirements:** Windows 10 version 2004+ (Build 19041+), 64-bit, hardware virtualization enabled in BIOS (VT-x/AMD-V).

#### macOS
1. Download Docker Desktop: https://www.docker.com/products/docker-desktop/
   - Apple Silicon (M1/M2/M3): download the Apple chip version
   - Intel Mac: download the Intel chip version
2. Drag Docker.app to Applications
3. Open Docker Desktop — authorize when asked
4. Wait for it to say "Docker is running"
5. Open Terminal and verify:
   ```
   docker --version
   docker compose version
   ```

#### Linux (Ubuntu/Debian)
```bash
# Install Docker Engine
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Let your user run Docker without sudo
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
```

### 2. Git (REQUIRED)

Git is needed to clone the repository and collaborate with your team.

#### Windows
Download and install Git for Windows: https://git-scm.com/download/win
(Accept all default settings during installation)

#### macOS
```bash
# If you have Homebrew:
brew install git

# Or install Xcode Command Line Tools (includes Git):
xcode-select --install
```

#### Linux
```bash
sudo apt install -y git
```

Verify:
```bash
git --version
```

### 3. OpenAI API Key (REQUIRED unless using Ollama)

The chatbot uses OpenAI's GPT-4o-mini for generating answers and text-embedding-3-small for creating vector embeddings.

1. Go to https://platform.openai.com/signup and create an account
2. Go to https://platform.openai.com/api-keys
3. Click "Create new secret key"
4. Copy the key (starts with `sk-`) — you'll paste it into the `.env` file later
5. Add billing: https://platform.openai.com/account/billing — you need at least $5 credit

**Cost estimate:** For a university chatbot, expect $5–15/month (about $0.15 per 1 million input tokens).

**Alternative: No API key needed if using Ollama** (runs LLMs locally on your machine, but requires 8GB+ RAM and ideally a GPU).

### 4. A Text Editor (RECOMMENDED)

You'll need to edit the `.env` file. Any text editor works:
- **VS Code** (free, recommended): https://code.visualstudio.com/
- Notepad++ (Windows)
- nano/vim (Linux/macOS terminal)

---

## Complete Setup: From Zero to Running Chatbot

### Phase 1: Get the Project Files

```bash
# Option A: Clone from Git repository
git clone https://github.com/your-org/chatbot-bcetd.git
cd chatbot-bcetd

# Option B: If you downloaded the files as a ZIP
# Unzip, then open a terminal/PowerShell in the chatbot-bcetd folder
cd chatbot-bcetd
```

After this step you should be inside a folder that looks like this:
```
chatbot-bcetd/
├── docker-compose.yml       ← This file is the key
├── .env.example
├── scripts/
├── member2-rag-pipeline/
├── member3-ethics-guardrails/
├── member4-analytics/
├── member5-frontend/
├── ...
```

### Phase 2: Configure Environment

```bash
# Copy the template to create your configuration file
cp .env.example .env
```

Now open `.env` in your text editor and change this line:
```
OPENAI_API_KEY=sk-your-api-key-here
```
Replace `sk-your-api-key-here` with your actual OpenAI API key.

**Also change the default passwords if deploying publicly:**
```
N8N_PASSWORD=your-secure-password-here
POSTGRES_PASSWORD=your-secure-password-here
```

Save the file.

### Phase 3: Start the System

#### On Linux / macOS:
```bash
chmod +x scripts/start.sh
./scripts/start.sh
```

#### On Windows (PowerShell):
```powershell
# The bash script won't work directly on Windows
# Use docker compose directly instead:
docker compose up -d --build
```

#### What Happens When You Run This:
1. Docker downloads all container images (first run only, ~2-3 GB total)
2. PostgreSQL starts and auto-creates the analytics database + tables
3. Qdrant starts (empty — no documents ingested yet)
4. n8n starts and becomes accessible at port 5678
5. The frontend (nginx) builds and starts at port 3000

**First run takes 3-10 minutes** (downloading images). Subsequent starts take 10-30 seconds.

### Phase 4: Verify Everything is Running

```bash
# Check all containers are up
docker compose ps
```

You should see something like:
```
NAME              STATUS         PORTS
bcetd-n8n         Up (healthy)   0.0.0.0:5678->5678/tcp
bcetd-qdrant      Up (healthy)   0.0.0.0:6333->6333/tcp
bcetd-postgres    Up (healthy)   0.0.0.0:5432->5432/tcp
bcetd-frontend    Up             0.0.0.0:3000->3000/tcp
```

Now open your browser:

| URL | What You'll See |
|-----|----------------|
| http://localhost:3000 | Student chat interface (ULBS branded) |
| http://localhost:3000/admin | Admin analytics dashboard |
| http://localhost:5678 | n8n workflow editor (login: admin / changeme_n8n) |
| http://localhost:6333/dashboard | Qdrant vector database dashboard |

### Phase 5: Import n8n Workflows

This is the most important manual step — loading the AI logic into n8n.

1. Open **http://localhost:5678** in your browser
2. Log in with username `admin` and the password from your `.env` (default: `changeme_n8n`)
3. You'll see an empty n8n workspace

**Add OpenAI Credentials first:**
1. Click the **☰** menu → **Credentials**
2. Click **Add Credential** → search for "OpenAI"
3. Paste your OpenAI API key → Save

**Add PostgreSQL Credentials:**
1. Click **Add Credential** → search for "Postgres"
2. Fill in:
   - Host: `postgres` (this is the Docker container name, NOT localhost)
   - Port: `5432`
   - Database: `chatbot_stats`
   - User: `chatbot_user`
   - Password: (the POSTGRES_PASSWORD from your `.env`)
3. Click **Test Connection** → should show "Connection successful"
4. Save

**Import the workflows (in this exact order):**

1. Click **☰** → **Workflows** → **Import from File**
2. Import these files one by one:

| Order | File to Import | After Import |
|-------|---------------|--------------|
| 1 | `member3-ethics-guardrails/workflow_guardrails.json` | Note the workflow ID from the URL bar |
| 2 | `member3-ethics-guardrails/workflow_output_validation.json` | Note the workflow ID |
| 3 | `member4-analytics/workflow_analytics_logging.json` | Note the workflow ID |
| 4 | `member2-rag-pipeline/workflow_ingestion_pipeline.json` | Just import |
| 5 | `member2-rag-pipeline/workflow_query_pipeline.json` | **Edit this one** (see below) |
| 6 | `member4-analytics/workflow_daily_stats.json` | Toggle to **Active** after import |
| 7 | `member4-analytics/workflow_analytics_api.json` | Toggle to **Active** after import |

**Editing the Query Pipeline (step 5):**
After importing the query pipeline, open it and find these nodes:
- "Ethics Guardrails (Sub-Workflow)" → click it → change the Workflow ID to the ID you noted in step 1
- "Output Validation (Layer 3)" → click it → change the Workflow ID to the ID from step 2
- All OpenAI nodes → make sure they're connected to your OpenAI credential
- Save the workflow
- Toggle it to **Active** (this enables the webhook that the chat UI calls)

### Phase 6: Add University Documents

Place your ULBS PDF, DOCX, or TXT files in the `data/documents/` folder:

```bash
# Example: copy some PDFs into the documents folder
cp ~/Downloads/calendarul_academic.pdf data/documents/
cp ~/Downloads/regulament_licenta.pdf data/documents/
cp ~/Downloads/taxe_scolarizare.pdf data/documents/
```

Then run the ingestion pipeline:
1. Open the "Document Ingestion Pipeline" workflow in n8n
2. Click the **Execute Workflow** button (play button)
3. Wait for it to complete — you'll see green checkmarks on each node

Verify documents were ingested:
```bash
curl http://localhost:6333/collections/ulbs_documents
```
You should see `"vectors_count": <some number greater than 0>`.

### Phase 7: Test the Chatbot

1. Open **http://localhost:3000**
2. Type a question about your documents (e.g., "Care este calendarul academic?")
3. You should get an answer with source citations

If you see "Nu am găsit informații suficiente" — that's the fallback message. It means either:
- Your documents don't contain the answer (working correctly)
- The ingestion pipeline hasn't run yet (go back to Phase 6)

---

## Common Commands Reference

```bash
# Start all core services
docker compose up -d

# Start with local LLM (Ollama)
docker compose --profile ollama up -d

# Start with Grafana monitoring
docker compose --profile monitoring up -d

# Start everything
docker compose --profile ollama --profile monitoring up -d

# Stop all services (keeps data)
docker compose --profile ollama --profile monitoring down

# View logs of a specific service
docker logs bcetd-n8n --tail 50 -f
docker logs bcetd-postgres --tail 50 -f
docker logs bcetd-qdrant --tail 50 -f

# Restart a single service
docker compose restart n8n

# Check which services are running
docker compose ps

# Full reset — DELETE ALL DATA
docker compose --profile ollama --profile monitoring down -v

# Rebuild frontend after changing HTML
docker compose up -d --build frontend
```

---

## Using Ollama Instead of OpenAI (Fully Offline)

If you don't want to use OpenAI (no API key, no cloud, 100% local):

```bash
# 1. Start with Ollama profile
docker compose --profile ollama up -d

# 2. Download the LLM models (first time only — 4-8 GB downloads)
docker exec bcetd-ollama ollama pull llama3.1
docker exec bcetd-ollama ollama pull nomic-embed-text

# 3. Verify models are downloaded
docker exec bcetd-ollama ollama list
```

Then in the n8n workflows, you need to swap OpenAI nodes for Ollama nodes:
- In the ingestion pipeline: disable "OpenAI Embeddings", enable "Ollama Embeddings (Local Alternative)"
- In the query pipeline: replace the OpenAI LLM node with an Ollama Chat node pointing to `http://ollama:11434`
- Change Qdrant vector size from 1536 to 384 in both workflows
- **Re-run the ingestion pipeline** after switching (old embeddings are incompatible)

---

## Troubleshooting

### "Docker is not running" or "Cannot connect to Docker daemon"
- **Windows/macOS:** Open Docker Desktop and wait for it to fully start
- **Linux:** Run `sudo systemctl start docker`

### "Port 5678 is already in use"
Something else is using that port. Either:
- Stop the other program, or
- Change the port in `.env`: `N8N_PORT=5679` and restart

### "Connection refused" when the Chat UI tries to send a message
The n8n query pipeline webhook isn't active. Go to n8n → open the query pipeline workflow → toggle it to **Active**.

### First start is very slow
Docker is downloading container images (PostgreSQL ~200MB, n8n ~800MB, Qdrant ~100MB, etc.). This only happens once. Check progress with:
```bash
docker compose logs -f
```

### "OPENAI_API_KEY is not set"
Edit your `.env` file and add your API key. Then restart:
```bash
docker compose down
docker compose up -d
```

### The chatbot answers "Nu am găsit informații" for everything
The Qdrant database is empty. You need to:
1. Put documents in `data/documents/`
2. Run the ingestion pipeline in n8n

### n8n workflows show errors about missing credentials
Go to n8n → Credentials → make sure OpenAI and PostgreSQL credentials are configured with the correct values.

---

## System Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| **RAM** | 4 GB | 8 GB (16 GB if using Ollama) |
| **Disk** | 5 GB free | 20 GB (includes Ollama models) |
| **CPU** | 2 cores | 4+ cores |
| **GPU** | Not needed | NVIDIA GPU for Ollama (much faster) |
| **OS** | Windows 10+, macOS 12+, Ubuntu 20.04+ | Any OS that runs Docker Desktop |
| **Network** | Internet for OpenAI API | Not needed if using Ollama |

---

## Folder Structure Explained

Everything lives in **one directory**:

```
chatbot-bcetd/                  ← Project root (you are here)
│
├── docker-compose.yml          ← Defines all 6 Docker containers
├── .env                        ← Your secrets (API keys, passwords)
├── .env.example                ← Template for .env
├── .gitignore                  ← Files Git should ignore
│
├── scripts/
│   ├── start.sh                ← One-command startup (Linux/macOS)
│   └── healthcheck.sh          ← Check if services are healthy
│
├── data/
│   └── documents/              ← PUT YOUR ULBS DOCUMENTS HERE
│
├── member2-rag-pipeline/       ← AI pipeline workflow files (import into n8n)
├── member3-ethics-guardrails/  ← Safety system workflow files (import into n8n)
├── member4-analytics/          ← Database schema + analytics workflows
├── member5-frontend/           ← Chat UI + Admin dashboard (auto-built by Docker)
├── member1-infrastructure/     ← Grafana config
├── shared/                     ← API contract documentation
├── docs/                       ← Team documentation
└── .github/workflows/          ← CI pipeline for GitHub
```

**You don't need to go inside most of these folders.** The main things you interact with are:
1. `.env` — edit your API key and passwords
2. `data/documents/` — put your university documents here
3. `docker compose` commands — start and stop the system
4. `http://localhost:5678` — n8n for importing and managing workflows
5. `http://localhost:3000` — the chatbot interface
