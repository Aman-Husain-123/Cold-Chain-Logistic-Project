# 🧊 Cold-Chain Logistics — FDE Agentic Intelligence Platform

> **Full-stack Data Engineering project** combining a live SQL telemetry database, a Pinecone vector knowledge base, and a LangGraph multi-tool AI agent, all surfaced through a Streamlit enterprise dispatch console.

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [System Architecture](#-system-architecture)
- [Tech Stack](#-tech-stack)
- [Project Structure](#-project-structure)
- [Environment Configuration](#-environment-configuration)
- [Setup Guide (Phase by Phase)](#-setup-guide-phase-by-phase)
  - [Phase 0 — Legacy Data Ingestion (SQL Server)](#phase-0--legacy-data-ingestion-sql-server)
  - [Phase 1 — SOP Knowledge Base Ingestion (Pinecone)](#phase-1--sop-knowledge-base-ingestion-pinecone)
  - [Phase 2 — Database Security & Semantic View](#phase-2--database-security--semantic-view)
  - [Phase 3 — Agent Testing (CLI)](#phase-3--agent-testing-cli)
  - [Phase 4 — Audit Logging](#phase-4--audit-logging)
  - [Phase 5 — Streamlit UI](#phase-5--streamlit-ui)
  - [Phase 6 — Cloud Deployment (AWS EC2)](#phase-6--cloud-deployment-aws-ec2)
- [Agent Tools](#-agent-tools)
- [LangGraph Orchestrator](#-langgraph-orchestrator)
- [Streamlit UI Features](#-streamlit-ui-features)
- [Data & Schema](#-data--schema)
- [Security Model](#-security-model)
- [LLM & Embedding Modes](#-llm--embedding-modes)
- [SOP Policy Document](#-sop-policy-document)
- [Sample Queries](#-sample-queries)
- [Requirements](#-requirements)

---

## 🧭 Project Overview

This project simulates a **production-grade, enterprise-level Full-stack Data Engineering (FDE)** pipeline for a logistics company operating cold-chain freight shipments. The system enables business dispatchers to **chat with their real-time fleet data** using natural language, powered by an autonomous AI agent.

### What it does

| Capability | Description |
|---|---|
| 🗄️ **Live Fleet Telemetry** | Queries a real MS SQL Server database containing IoT sensor data (temperature, GPS, cargo condition) via a secure read-only view |
| 🌦️ **Live Corridor Conditions** | Fetches real-time weather data (temperature, wind speed, congestion index) for any GPS coordinate via a public REST API |
| 📚 **Compliance SOP Retrieval** | Semantically searches enterprise Standard Operating Procedures stored in Pinecone vector database |
| 🤖 **Autonomous AI Reasoning** | A LangGraph ReAct-style agent decides which tools to call, in what order, to answer dispatcher queries end-to-end |
| 🖥️ **Enterprise Dispatch UI** | Streamlit frontend with live streaming agent traces, tool input/output inspection, and an admin audit log viewer |
| 🛡️ **Agent Audit Trail** | Every tool call and LLM response is logged to a dedicated SQL audit table for full traceability and compliance |

---

## 🏗️ System Architecture

```
+------------------------------------------------------------------------+
|                        STREAMLIT DISPATCH CONSOLE                      |
|                           (src/ui.py)                                  |
+------------------------------+-----------------------------------------+
                               |  User Query (Natural Language)
                               v
+----------------------------------------------------------------------+
|                    LANGGRAPH ORCHESTRATOR                            |
|                      (src/orchestrator.py)                           |
|                                                                      |
|   +--------------+    +------------------------------------------+  |
|   |  AgentState  |<-->|  Reasoning Node (LLM with bound tools)   |  |
|   |  (Memory)    |    +--------------+---------------------------+  |
|   +--------------+                   |  tool_calls                  |
|                                      v                               |
|                    +-----------------------------+                   |
|                    |      ToolNode (Executor)    |                   |
|                    +------+----------+-----------+                   |
+----------------------------+----------+-------------------------------|
                             |          |
           +-----------------+          +---------------------+
           v                                                   v
+---------------------+              +------------------------------------+
|  Tool 1             |              |  Tool 2                            |
|  query_telemetry_db |              |  fetch_corridor_conditions         |
|                     |              |                                    |
|  SQL Server via     |              |  Open-Meteo REST API               |
|  ODBC Driver 18     |              |  (Live weather + wind speed)       |
|  FDE_VIEWS.VW_      |              |                                    |
|  ACTIVE_FLEET       |              +------------------------------------+
+---------------------+
           v
+---------------------+              +------------------------------------+
|  Tool 3             |              |  Audit Logging                     |
|  search_compliance_ |              |                                    |
|  sop                |              |  FDE_VIEWS.AgentAuditLog           |
|                     |              |  (SQL Server — INSERT only)        |
|  Pinecone Vector DB |              |                                    |
|  (SOP documents)    |              +------------------------------------+
+---------------------+
```

### Agent Decision Flow

```
User Query --> Reasoner (LLM) --> Tool Call Decision
                   ^                      |
                   |              ToolNode Executes
                   |                      |
                   +---- Observation <----+
                   (loop until no more tool calls)
                         |
                         v
                  Final Response to User
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Agent Framework** | LangGraph 1.2.x (StateGraph, MemorySaver, ToolNode) |
| **LLM Options** | OpenAI `gpt-4o`, DeepSeek `deepseek-v4-flash/pro`, Ollama `qwen2.5:7b` |
| **Embeddings Options** | OpenAI `text-embedding-ada-002` (1536d), HuggingFace `BAAI/bge-m3` (1024d) |
| **Vector Database** | Pinecone (Serverless, AWS `us-east-1`) |
| **Relational Database** | Microsoft SQL Server 2022 (via Docker) |
| **ORM / DB Driver** | SQLAlchemy + PyODBC (ODBC Driver 18) |
| **UI Framework** | Streamlit 1.62+ |
| **Live Weather API** | Open-Meteo (free, no API key required) |
| **Language** | Python 3.12.13 |

---

## 📁 Project Structure

```
cold-chain-logistics-FDE-Project/
│
├── src/                          # Core application source code
│   ├── agent_tools.py            # LangChain @tool definitions (3 tools)
│   ├── orchestrator.py           # LangGraph StateGraph agent
│   ├── ui.py                     # Streamlit enterprise dispatch console
│   └── prompts/
│       └── system_prompt.txt     # Business-structured LLM system prompt
│
├── scripts/                      # One-time setup & ingestion scripts
│   ├── ingest_legacy_data.py     # Loads CSV into MS SQL Server (Phase 0)
│   ├── ingest_sop_pinecone.py    # Loads SOP docs into Pinecone (Phase 1)
│   └── setup_security_and_view.sql  # Creates SQL view, roles, permissions (Phase 2)
│
├── data/
│   ├── raw/
│   │   └── dynamic_supply_chain_logistics_dataset.csv   # ~15 MB source dataset
│   ├── policy/
│   │   └── Cold_Chain_Incident_SOP_v2.md                # Enterprise SOP document
│   ├── cache/
│   │   └── ingestion_hash_cache.json                    # Incremental Pinecone sync cache
│   └── source/
│       └── data.txt                                     # Dataset download link
│
├── docs/
│   └── instrutions.md            # Phase-by-phase developer setup guide
│
├── .streamlit/
│   └── config.toml               # Streamlit server configuration
│
├── .github/
│   └── workflows/                # GitHub Actions CI/CD workflows
│
├── .gitignore
├── requirements.txt              # Python dependencies (Python 3.12.13)
└── notes.pdf                     # Project reference notes
```

---

## ⚙️ Environment Configuration

Create a `.env` file at the **project root** with the following variables:

```env
# LLM Selection: OPENAI | DEEPSEEK | OLLAMA (default: OLLAMA)
Agent_llm=DEEPSEEK

# Embeddings Selection: OPENAI | LOCAL (default: LOCAL)
Embeddings_model=LOCAL

# Local model name (only used if Embeddings_model=LOCAL)
Local_Embedding_Model=BAAI/bge-m3

# API Keys
PINECONE_API_KEY=your_pinecone_api_key_here
OPENAI_API_KEY=your_openai_api_key_here
DEEPSEEK_API_KEY=your_deepseek_api_key_here

# SQL Server (Legacy Database)
SQL_SERVER_HOST=localhost
SQL_SERVER_PORT=1433

# Admin user (used only during data ingestion scripts)
SQL_ADMIN_USER=sa
SQL_ADMIN_PASSWORD=FdeEnterprisePass123!

# Read-only agent user (used by the AI agent at runtime)
SQL_AGENT_USER=USR_FDE_RO
SQL_AGENT_PASSWORD=AgentPassword2026!
```

> **Note:** Never commit your `.env` file. It is already listed in `.gitignore`.

---

## 🚀 Setup Guide (Phase by Phase)

### Phase 0 — Legacy Data Ingestion (SQL Server)

**Step 1: Download the dataset**

Download `dynamic_supply_chain_logistics_dataset.csv` (link in `data/source/data.txt`) and place it at:
```
data/raw/dynamic_supply_chain_logistics_dataset.csv
```

**Step 2: Spin up MS SQL Server 2022 via Docker**

```bash
# Local machine
docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=FdeEnterprisePass123!" -p 1433:1433 --name legacy-mssql -d mcr.microsoft.com/mssql/server:2022-latest

# AWS EC2 with persistent volume (recommended)
docker run -v mssql_data:/var/opt/mssql \
  -e "ACCEPT_EULA=Y" \
  -e "MSSQL_SA_PASSWORD=FdeEnterprisePass123!" \
  -p 1433:1433 \
  --name legacy-mssql \
  -d mcr.microsoft.com/mssql/server:2022-latest
```

**Step 3: Install dependencies**

```bash
pip install -r requirements.txt
```

**Step 4: Ingest the CSV into SQL Server**

```bash
python scripts/ingest_legacy_data.py
```

This maps clean column names to a legacy enterprise schema and ingests them into `dbo.TBL_SC_FLEET_HIST_RAW`:

| Clean Column | Legacy Column |
|---|---|
| `timestamp` | `TS_UTC` |
| `vehicle_gps_latitude` | `V_LAT` |
| `vehicle_gps_longitude` | `V_LON` |
| `iot_temperature` | `IOT_TEMP_VAL_C` |
| `cargo_condition_status` | `CGO_COND_CD` |
| `risk_classification` | `RISK_CLS_TXT` |
| `delay_probability` | `DELAY_PROB_DEC` |
| `port_congestion_level` | `PRT_CNG_LVL` |
| `route_risk_level` | `RT_RSK_IDX` |

**Connecting with VS Code (SQL Server extension)**

Install the **"SQL Server (mssql)"** extension by Microsoft, then add a connection:

```
Profile Name:             legacy-mssql
Server name:              localhost
Port:                     1433
Authentication type:      SQL Login
User name:                sa
Password:                 FdeEnterprisePass123!
Encrypt:                  Optional (or False)
Trust server certificate: ON (crucial for Docker)
```

---

### Phase 1 — SOP Knowledge Base Ingestion (Pinecone)

**Step 1:** Get your API key from https://app.pinecone.io/ and add it to `.env` as `PINECONE_API_KEY`.

**Step 2:** Run the ingestion script:

```bash
python scripts/ingest_sop_pinecone.py
```

The script:
- **Auto-provisions** a Pinecone serverless index (AWS `us-east-1`) with the correct dimension for your embedding model choice
- **Parses multiple formats**: `.md`, `.txt`, `.pdf`, `.csv`, `.xlsx`
- **Incremental sync**: Uses MD5 hash caching — re-running only re-processes changed or new files
- **Self-healing**: Detects and purges mismatched dimension indexes automatically
- **Batch upserts** in configurable batches of 100 vectors

| Embedding Mode | Index Name | Dimensions |
|---|---|---|
| `OPENAI` | `fde-sop-index-openai` | 1536 |
| `LOCAL` (BGE-M3) | `fde-sop-index-local` | 1024 |

---

### Phase 2 — Database Security & Semantic View

Run `scripts/setup_security_and_view.sql` against your MS SQL Server using the `sa` connection.

This script:

1. **Creates `FDE_VIEWS` schema** — an isolated namespace for AI-accessible objects
2. **Creates a semantic view** `FDE_VIEWS.VW_ACTIVE_FLEET` — translates cryptic legacy column names to clean English for the LLM
3. **Creates a read-only SQL login** `USR_FDE_RO` used by the agent at runtime
4. **Enforces least-privilege access**:
   - GRANT SELECT ON FDE_VIEWS.VW_ACTIVE_FLEET TO USR_FDE_RO
   - DENY SELECT ON dbo.TBL_SC_FLEET_HIST_RAW TO USR_FDE_RO
   - DENY INSERT, UPDATE, DELETE, ALTER ON SCHEMA::dbo TO USR_FDE_RO

**Verify the security setup:**

```sql
-- This SHOULD work (agent has access to clean view)
SELECT TOP 5 * FROM FDE_VIEWS.VW_ACTIVE_FLEET;

-- This SHOULD fail (agent is blocked from raw legacy table)
SELECT TOP 5 * FROM dbo.TBL_SC_FLEET_HIST_RAW;
```

---

### Phase 3 — Agent Testing (CLI)

```bash
# Test all three tools individually
python src/agent_tools.py

# Run the full CLI dispatch loop
python src/orchestrator.py
```

**Sample test queries:**

Query 1 — The "Domino Effect" Test (triggers all 3 tools):
```
Find any active shipments near Los Angeles (Latitude ~33.8, Longitude ~-118.1).
Check the local weather there, and tell me if the current cargo temperature
violates the SOP for fresh perishables.
```

Query 2 — The "Restraint" Test (LLM answers from knowledge, no tools):
```
I am a new dispatcher on the night shift. Can you quickly explain the
difference between a Tier 1 and Tier 2 escalation?
```

---

### Phase 4 — Audit Logging

Run this SQL against the `sa` connection:

```sql
-- Create the audit log table
CREATE TABLE FDE_VIEWS.AgentAuditLog (
    LogID        INT IDENTITY(1,1) PRIMARY KEY,
    Timestamp    DATETIME DEFAULT GETDATE(),
    SessionID    VARCHAR(50),
    NodeExecuted VARCHAR(50),
    ToolName     VARCHAR(100),
    Content      NVARCHAR(MAX)
);

-- Grant the agent write permission ONLY to this table
GRANT INSERT ON FDE_VIEWS.AgentAuditLog TO USR_FDE_RO;
```

Every tool invocation and LLM response is then automatically logged with:
- **SessionID** — unique UUID per browser session
- **NodeExecuted** — which LangGraph node ran (`reasoner`, `tools`, `reasoner_final`)
- **ToolName** — the specific tool triggered
- **Content** — the raw JSON arguments or text response

---

### Phase 5 — Streamlit UI

```bash
streamlit run src/ui.py
```

Opens at `http://localhost:8501`.

---

### Phase 6 — Cloud Deployment (AWS EC2)

**Recommended EC2 spec:** `c7i-flex.large`, 30 GB storage, Ubuntu (Linux)

```bash
# SSH into EC2 and install dependencies
sudo apt update && sudo apt install -y python3-pip python3-venv git

# Clone and set up the project
cd /home/ubuntu
git clone https://github.com/nimowhyca/cold-chain-logistics-FDE-Project.git
cd cold-chain-logistics-FDE-Project
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Install Microsoft ODBC Driver 18
curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | sudo gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg
curl -fsSL https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/prod.list | sudo tee /etc/apt/sources.list.d/mssql-release.list
sudo apt-get update
sudo ACCEPT_EULA=Y apt-get install -y msodbcsql18
sudo apt-get install -y unixodbc-dev
```

**Create a systemd service for auto-restart:**

```bash
sudo nano /etc/systemd/system/streamlit.service
```

Paste:

```ini
[Unit]
Description=Streamlit Cold-Chain Dispatch Console
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/cold-chain-logistics-FDE-Project
ExecStart=/home/ubuntu/cold-chain-logistics-FDE-Project/venv/bin/streamlit run src/ui.py --server.port=8501 --server.address=0.0.0.0
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable streamlit
sudo systemctl start streamlit
```

The app will be accessible at `http://<EC2_PUBLIC_IP>:8501` and restarts automatically on reboot.

> **Note:** Ensure your EC2 Security Group allows inbound traffic on ports `8501` and `1433`.

---

## 🔧 Agent Tools

Defined in `src/agent_tools.py`. These are the three `@tool`-decorated functions the LangGraph agent can call:

### 1. `query_telemetry_db(sql_query: str) -> str`

Executes a T-SQL `SELECT` query against `FDE_VIEWS.VW_ACTIVE_FLEET`.

- **Security:** Hard-blocks any non-`SELECT` statement before execution
- **Returns:** Up to 10 rows with column headers
- **Columns:** `Timestamp`, `Latitude`, `Longitude`, `Current_Temperature_C`, `Cargo_Condition_Code`, `Risk_Classification`, `Delay_Probability`, `Port_Congestion_Level`, `Route_Risk_Index`

### 2. `fetch_corridor_conditions(latitude: float, longitude: float) -> str`

Fetches live weather data from the **Open-Meteo API** (free, no API key required).

- **Returns:** External temperature, wind speed (km/h), and a computed corridor congestion index
- **Logic:** Wind speed > 10 km/h triggers `Congestion Index: 8.5/10` (High Transit Disruption)

### 3. `search_compliance_sop(query: str) -> str`

Performs a semantic similarity search over SOP documents indexed in Pinecone.

- **Returns:** Top-2 most relevant document chunks with source file metadata
- **Use case:** Regulatory thresholds, breach mitigation steps, escalation rules

---

## 🧠 LangGraph Orchestrator

Defined in `src/orchestrator.py`.

### Graph Architecture

```
START
  |
  v
[reasoner node]  <--------------------------+
  |                                          |
  | tool_calls?                              |
  +-- YES --> [tools node] --> (observation) +
  |
  +-- NO --> END
```

- **`AgentState`**: A `TypedDict` holding the full message history with `add_messages` reducer for proper state accumulation
- **`MemorySaver`**: In-memory checkpointer enabling multi-turn conversation per `thread_id`
- **`tools_condition`**: Built-in conditional edge that routes to `ToolNode` if tool calls are present, or to `END` for final responses

### LLM Configuration (set via `.env`)

| `Agent_llm` value | Model | Notes |
|---|---|---|
| `OPENAI` | `gpt-4o` | Cloud, best reasoning |
| `DEEPSEEK` | `deepseek-v4-flash` or `deepseek-v4-pro` | Cloud, cost-efficient |
| `OLLAMA` (default) | `qwen2.5:7b` | Local, no API cost |

---

## 🖥️ Streamlit UI Features

Located in `src/ui.py`.

### 🧊 Dispatch Console (Tab 1)

- **Chat interface** with streaming agent execution updates
- **Live status indicator** showing which LangGraph node is executing
- **Tool call transparency**: Expandable panels showing exact arguments sent to each tool and raw output returned
- **Multi-session support**: Each browser tab gets a unique `thread_id` (UUID) with isolated conversation memory
- **Session purge button**: Clears conversation state and starts fresh

### 🛡️ Security & Audit Logs (Tab 2)

- **Admin authentication gate**: Requires `SQL_ADMIN_USER` / `SQL_ADMIN_PASSWORD` credentials to access
- **Full audit trail viewer**: Displays all rows from `FDE_VIEWS.AgentAuditLog` in a formatted, sortable dataframe
- **Columns**: Log ID, Execution Time, Session Token, Graph Node, Tool Triggered, Raw Payload Data

### Enterprise Theme

- Dark mode background (`#0B0E14`)
- Custom sidebar styling (`#111622`)
- Code block accent color (`#38BDF8`)

---

## 📊 Data & Schema

### Source Dataset

**File:** `data/raw/dynamic_supply_chain_logistics_dataset.csv` (~15 MB)

| Field | Description |
|---|---|
| `timestamp` | UTC timestamp of the telemetry record |
| `vehicle_gps_latitude` | Vehicle latitude coordinate |
| `vehicle_gps_longitude` | Vehicle longitude coordinate |
| `iot_temperature` | IoT sensor temperature reading (degrees C) |
| `cargo_condition_status` | Cargo condition code |
| `risk_classification` | Risk classification text (e.g., `High Risk`) |
| `delay_probability` | Probability of delivery delay (0.0 to 1.0) |
| `port_congestion_level` | Port congestion severity index (0.0 to 10.0) |
| `route_risk_level` | Route risk index |

### SQL View (Agent-facing)

```sql
SELECT
    Timestamp, Latitude, Longitude,
    Current_Temperature_C, Cargo_Condition_Code,
    Risk_Classification, Delay_Probability,
    Port_Congestion_Level, Route_Risk_Index
FROM dbo.TBL_SC_FLEET_HIST_RAW;
```

---

## 🔐 Security Model

This project implements a **defense-in-depth** security architecture for AI agents:

```
THREAT: LLM hallucination -> destructive SQL

DEFENSES:
  1. Agent runs as USR_FDE_RO (read-only SQL login)
  2. Agent can only SELECT from VW_ACTIVE_FLEET
  3. Raw table dbo.TBL_SC_FLEET_HIST_RAW is DENIED
  4. INSERT/UPDATE/DELETE/ALTER on dbo schema DENIED
  5. Tool code hard-blocks non-SELECT queries at runtime
  6. Audit log captures every tool call for review
```

### Privilege Matrix

| User | Raw Table (dbo) | Semantic View | Audit Log |
|---|---|---|---|
| `sa` (admin) | Full access | Full access | Full access |
| `USR_FDE_RO` (agent) | DENIED | SELECT only | INSERT only |

---

## 🤖 LLM & Embedding Modes

The project supports **hot-swappable** LLM and embedding backends via the `.env` file — no code changes needed.

### Agent LLM Options (`Agent_llm`)

```
OPENAI   -> gpt-4o via langchain_openai (requires OPENAI_API_KEY)
DEEPSEEK -> deepseek-v4-flash via OpenAI-compatible endpoint (requires DEEPSEEK_API_KEY)
OLLAMA   -> qwen2.5:7b via local Ollama server (no API key needed, default)
```

### Embedding Options (`Embeddings_model`)

```
OPENAI -> text-embedding-ada-002 (1536 dimensions, requires OPENAI_API_KEY)
LOCAL  -> BAAI/bge-m3 via HuggingFace (1024 dimensions, runs on CPU, default)
```

> **Performance tip:** When running Streamlit, the local HuggingFace model is loaded once and permanently cached in RAM using `@st.cache_resource`, preventing re-initialization on every script rerun.

---

## 📜 SOP Policy Document

Located at `data/policy/Cold_Chain_Incident_SOP_v2.md`.

**Standard Operating Procedure: Cold-Chain & Transit Anomalies** (Version 2.4, Jan 2021)

| Rule | Threshold | Action |
|---|---|---|
| **Fresh Perishables Temperature** | 0.0C to 4.0C required | If temperature > 4.0C: Declare breach, restart auxiliary cooling unit. If ETA delay > 1 hour: divert to nearest emergency cold-storage. |
| **Port Congestion** | Congestion index > 7.0 | Suspend standard routing. Divert all active shipments to **Inland Empire Overflow Depot (San Bernardino)** for cross-docking. |
| **High Risk Escalation** | Risk = "High Risk" AND Delay Probability > 0.65 | Escalate to **Tier 2 Logistics Manager** immediately. |

---

## 💬 Sample Queries

Try these in the Streamlit Dispatch Console:

**Multi-tool Domino Effect Query:**
```
Find any active shipments near Los Angeles (Latitude ~33.8, Longitude ~-118.1).
Check the local weather there, and tell me if the current cargo temperature
violates the SOP for fresh perishables.
```

**Pure reasoning query (no tool calls):**
```
I am a new dispatcher on the night shift. Can you quickly explain the
difference between a Tier 1 and Tier 2 escalation?
```

**Direct telemetry query:**
```
Show me all shipments with a delay probability greater than 0.8 and
a port congestion level above 5.
```

**SOP lookup:**
```
What is the mitigation protocol for a cold-chain breach on a fresh perishables shipment?
```

---

## 📦 Requirements

Python **3.12.13** is required. All dependencies are pinned in `requirements.txt`:

```
altair==6.2.2
faiss-cpu==1.15.0
langchain-community==0.4.2
langchain-huggingface==1.2.2
langchain-openai==1.6.0
langchain-pinecone==0.2.13
langgraph==1.2.11
langgraph-checkpoint==4.2.0
langsmith==0.11.1
openai==3.3.1
openpyxl==3.1.5
pandas==3.0.5
pinecone==7.3.0
pydantic-settings==2.15.0
pyodbc==5.3.0
pypdf==6.16.2
python-dotenv==1.2.3
scikit-learn==1.9.0
sentence-transformers==6.0.0
SQLAlchemy==2.0.52
streamlit==1.62.0
torch==2.13.0
```

> **System requirement:** ODBC Driver 18 for SQL Server must be installed on your OS.
> Download: https://learn.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server

---

*Built with LangGraph • Pinecone • Microsoft SQL Server • Streamlit*
