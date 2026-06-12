# Databricks Platform Learning Journal

> **Learner:** Director of Data Science  
> **Mentor:** Cursor Cloud Agent (Composer 2.5)  
> **Primary material:** [Databricks Apps Cookbook](https://github.com/databricks-solutions/databricks-apps-cookbook)  
> **Started:** 2026-06-11  
> **Approach:** Progressive, architecture-first, hands-on. One module at a time.

---

## How to use this journal

Each module adds a section below with concepts, diagrams, repo pointers, exercises, and a checkpoint. Answer the checkpoint questions before moving on — the next module builds on your answers.

| Module | Topic | Status |
|--------|-------|--------|
| 1 | Platform Architecture | **In progress** |
| 2 | Data Ingestion | Not started |
| 3 | Data Storage and Governance | Not started |
| 4 | Data Processing | Not started |
| 5 | Machine Learning | Not started |
| 6 | AI Applications | Not started |
| 7 | Production Operations | Not started |

---

# Learning Roadmap

This roadmap uses the cookbook as a **lens** on the full Databricks platform. The repo focuses heavily on **Module 6 (AI Applications)** and touches Modules 3–5. Modules 2 and parts of 4–7 are taught conceptually first, then connected to repo examples where they exist.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        YOUR LEARNING ARC                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  M1 Platform Architecture     ← You are here                          │
│       ↓                                                                  │
│  M2 Data Ingestion            (conceptual + workflow triggers in repo)  │
│       ↓                                                                  │
│  M3 Storage & Governance      (tables, volumes, UC — core repo patterns) │
│       ↓                                                                  │
│  M4 Data Processing           (jobs/workflows; medallion in practice)   │
│       ↓                                                                  │
│  M5 Machine Learning          (serving, vector search; not full MLflow)  │
│       ↓                                                                  │
│  M6 AI Applications           (Apps, Genie, dashboards — repo sweet spot)│
│       ↓                                                                  │
│  M7 Production Operations     (auth, secrets, cost, reliability)        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Module summaries (what you will learn)

| # | Module | Architectural focus | Cookbook coverage |
|---|--------|---------------------|-------------------|
| 1 | **Platform Architecture** | Workspace, control plane vs data plane, compute types, identity | `app.yaml`, SDK patterns, multi-framework layout |
| 2 | **Data Ingestion** | Landing zone → bronze; batch vs streaming; Auto Loader, DLT | Job triggers (`workflows_run`); table insert (`fastapi`) |
| 3 | **Storage & Governance** | Delta, Unity Catalog, Volumes, grants | `tables_read`, `volumes_upload`, `unity_catalog_get`, OBO |
| 4 | **Data Processing** | ETL/ELT, medallion, Workflows | `workflows_run`, `workflows_get_results`, cluster connect |
| 5 | **Machine Learning** | Features, experiments, registry, serving | `ml_serving_invoke`, `ml_vector_search` |
| 6 | **AI Applications** | Apps runtime, Genie, dashboards, MCP | Entire `streamlit/`, `reflex/`, `dash/`, `fastapi/` apps |
| 7 | **Production Operations** | Monitoring, cost, governance, secrets | `secrets_retrieve`, `external_connections`, `app.yaml` env |

### Recommended repo reading order (cross-module)

1. `docs/docs/intro.md` → `docs/docs/deploy.md`
2. `streamlit/views/tables_read.py` (data access foundation)
3. `streamlit/views/users_get_current.py` + `users_obo.py` (identity)
4. `streamlit/views/workflows_run.py` (orchestration touchpoint)
5. `streamlit/views/ml_serving_invoke.py` (ML in production)
6. `streamlit/app.py` + `view_groups.py` (full app map)

---

# Module 1: Platform Architecture

**Status:** In progress — awaiting your checkpoint answers  
**Estimated depth:** Foundation layer for everything else  
**Prerequisite knowledge assumed:** ML lifecycle, production systems, Python/SQL

---

## 1.1 Lesson — First principles

### What is Databricks, architecturally?

Databricks is a **managed data and AI platform** built around three ideas:

1. **Separation of storage and compute** — Data lives in cloud object storage (S3, ADLS, GCS) as Delta files. Compute spins up when needed and scales independently.
2. **A unified governance layer** — Unity Catalog sits above storage and compute, giving one place to define tables, permissions, lineage, and connections.
3. **Multiple personas, one platform** — Analysts (SQL), engineers (Spark pipelines), scientists (notebooks, MLflow), and app builders (Databricks Apps) all touch the same governed data.

Think of it as: **object storage + Delta + Unity Catalog + elastic compute + APIs** — not “a Spark cluster you rent.”

### Control plane vs data plane

```
┌──────────────────────────────────────────────────────────────────┐
│                     CONTROL PLANE (Databricks-managed)            │
│  • Workspace UI, REST APIs, Jobs scheduler                       │
│  • Unity Catalog metastore (metadata, grants, lineage)             │
│  • Identity: users, groups, service principals                     │
│  • Apps runtime orchestration                                      │
└────────────────────────────┬─────────────────────────────────────┘
                             │ APIs / auth tokens
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                     DATA PLANE (your cloud account)               │
│  • Object storage (Delta files, Volumes files)                   │
│  • SQL Warehouses, Clusters, Serverless compute                    │
│  • Lakebase database instances                                   │
│  • Network paths to external systems (connections)               │
└──────────────────────────────────────────────────────────────────┘
```

**Why this matters:** Your apps and jobs never “own” data. They request access through governed APIs. The cookbook recipes always go *through* workspace APIs (SDK, SQL connector) — they don't bypass governance.

### Core services (the map you will reuse)

| Service | Role | Analogy for a DS leader |
|---------|------|-------------------------|
| **Unity Catalog** | Catalog of tables, volumes, models, connections; permissions | Your enterprise data dictionary + ACLs |
| **Delta Lake** | Open table format on object storage (ACID, time travel) | The actual files your tables point to |
| **SQL Warehouse** | Managed compute for SQL/analytics | Fast, always-on query engine for apps and BI |
| **Clusters / Serverless** | Spark compute for transformation & ML training | Heavy lifting engine |
| **Workflows (Jobs)** | Scheduled/triggered pipelines | Airflow-like orchestration, native to platform |
| **Model Serving** | Hosted inference endpoints | Production model API |
| **Lakebase** | Managed PostgreSQL (OLTP) synced with lakehouse | App state, transactional workloads |
| **Databricks Apps** | Managed Python app hosting | Deploy Streamlit/FastAPI without building K8s |
| **Genie / AI·BI** | Natural-language analytics & dashboards | Consumer-facing intelligence layer |

This cookbook is primarily a tour of **how Apps talk to everything else**.

---

## 1.2 Lesson — How services interact (the request path)

When a user opens the Streamlit cookbook deployed as a Databricks App:

```
 User Browser
      │
      ▼
┌─────────────┐     identity headers        ┌──────────────────┐
│ Databricks  │ ──────────────────────────► │  Cookbook App    │
│ Apps proxy  │   (user token, email, etc.) │  (Streamlit)     │
└─────────────┘                             └────────┬─────────┘
                                                     │
                     ┌───────────────────────────────┼───────────────────────────────┐
                     │                               │                               │
                     ▼                               ▼                               ▼
              SQL Warehouse                   WorkspaceClient API              Model Serving
              (read Delta table)              (list catalogs, trigger job)     (invoke LLM)
                     │                               │                               │
                     └───────────────────────────────┴───────────────────────────────┘
                                                     │
                                                     ▼
                                            Unity Catalog
                                            (permissions enforced)
```

**Three integration patterns** appear everywhere in this repo:

| Pattern | When to use | Cookbook example |
|---------|-------------|------------------|
| **SQL Connector** | Query tabular data via warehouse | `streamlit/views/tables_read.py` |
| **Databricks SDK** (`WorkspaceClient`) | Control-plane operations: list resources, run jobs, call serving | `tables_read.py` (list warehouses), `workflows_run.py` |
| **Databricks Connect** | Run Spark code on remote compute | `streamlit/views/compute_connect_serverless.py` |

**Design insight:** SQL for *reading governed tables*, SDK for *platform operations*, Spark for *distributed transformation*. Apps rarely do heavy ETL — they consume curated outputs.

---

## 1.3 Lesson — Identity and authorization (why Apps feel different)

Databricks Apps run as a **service principal** by default. That identity has explicit grants on warehouses, tables, endpoints, etc.

Two modes matter for enterprise apps:

| Mode | Who is authenticated | Row-level security | Typical use |
|------|---------------------|-------------------|-------------|
| **Service principal** | The app itself | Uses app's grants only | Internal tools, trusted datasets |
| **On-behalf-of-user (OBO)** | The logged-in user | User's grants apply | Multi-tenant apps, sensitive data |

The cookbook makes this tangible in `streamlit/views/users_obo.py`:

- **OBO path:** reads `X-Forwarded-Access-Token` from request headers, passes it to the SQL connector
- **Service principal path:** uses `Config().authenticate` (the app's identity)

```python
# OBO — query runs as the human user
access_token=user_token

# Service principal — query runs as the app
credentials_provider=lambda: cfg.authenticate
```

**Why the platform is designed this way:** Apps are not trusted with user passwords. The Apps proxy injects short-lived tokens. Your code chooses whether to act as the app or pass through the user's identity. This is the same pattern as OAuth delegation in enterprise SaaS.

See also: `streamlit/views/users_get_current.py` — inspects headers to show who is calling the app.

---

## 1.4 Lesson — Common enterprise architectures

You will see three recurring patterns in mature Databricks deployments. The cookbook illustrates the **consumption layer** of each.

### Pattern A: Lakehouse-centric (most common)

```
Sources → Ingestion (Jobs/DLT) → Bronze → Silver → Gold (Delta/UC)
                                                      │
                                    ┌─────────────────┼─────────────────┐
                                    ▼                 ▼                 ▼
                              SQL Warehouse    Model Serving      Databricks Apps
                              (BI, SQL)        (ML/GenAI)         (custom UIs)
```

**Cookbook role:** Apps and serving read from Gold tables; jobs are triggered but not defined here.

### Pattern B: Lakehouse + Lakebase (transactional + analytical)

```
                    ┌── Delta tables (analytics, history)
Lakehouse (UC) ─────┤
                    └── Lakebase (OLTP, app state, low-latency CRUD)
                              │
                              └── FastAPI app reads/writes orders
```

**Cookbook role:** `fastapi/` demonstrates dual-database architecture — SQL warehouse for UC tables, Lakebase for transactional orders with OAuth token refresh (`fastapi/config/database.py`).

### Pattern C: AI application layer

```
Gold tables / Vector Index → Model Serving + Vector Search
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
              Custom App          Genie space      AI/BI Dashboard
              (RAG, agents)       (NL queries)     (embedded analytics)
```

**Cookbook role:** `ml_vector_search.py`, `genie_api.py`, `embed_dashboard.py`, `mcp_connect.py`.

---

## 1.5 Lesson — How this repository models the platform

The repo is **not** a full data platform implementation. It is a **reference consumption layer** — intentionally.

| Repo artifact | Architectural meaning |
|---------------|----------------------|
| `streamlit/`, `dash/`, `reflex/` | Same integration recipes, different UI frameworks — proves patterns are platform-level, not framework-specific |
| `fastapi/` | Headless/API consumption pattern for microservices |
| `app.yaml` (per app) | Apps runtime contract: entry command, env vars, resource bindings |
| `databricks.yml` | Asset Bundle stub for workspace-aware development |
| `docs/` | Published recipe docs with permissions and dependencies |
| `view_groups.py` | Maps recipe categories to platform capability areas |

### The `app.yaml` contract (read this file)

Example from `fastapi/app.yaml`:

```yaml
command: ["uvicorn", "app:app"]
env:
  - name: 'DATABRICKS_WAREHOUSE_ID'
    valueFrom: 'sql_warehouse'   # platform injects bound resource
```

**Design insight:** You declare *what* resources the app needs; Databricks injects connection details at deploy time. No hard-coded warehouse URLs in production.

---

## 1.6 Relevant repository files (Module 1 reading list)

| Priority | File | Read for |
|----------|------|----------|
| ★★★ | `docs/docs/intro.md` | Scope and framework choices |
| ★★★ | `docs/docs/deploy.md` | Local vs Apps deployment model |
| ★★★ | `streamlit/app.yaml` | Minimal Apps runtime definition |
| ★★★ | `fastapi/app.yaml` | Resource binding (warehouse, Lakebase env) |
| ★★ | `streamlit/views/tables_read.py` | SQL Connector + SDK together |
| ★★ | `streamlit/views/users_obo.py` | OBO vs service principal |
| ★★ | `streamlit/views/users_get_current.py` | Request identity in Apps |
| ★★ | `streamlit/views/compute_connect_serverless.py` | Databricks Connect / Spark path |
| ★ | `fastapi/app.py` | App lifecycle, conditional Lakebase init |
| ★ | `databricks.yml` | Bundle/workspace targeting |
| ★ | `streamlit/view_groups.py` | Full map of platform touchpoints |

---

## 1.7 Hands-on exercise — “Trace a query”

**Goal:** Experience the platform layers yourself, not just read about them.

### Part A — Local (30 min)

1. Clone/open this repo; `cd streamlit`
2. Create venv, `pip install -r requirements.txt`
3. `databricks auth login --host https://<your-workspace>/`
4. `export DATABRICKS_HOST=https://<your-workspace>/`
5. `streamlit run app.py`
6. Open **Tables → Read a Databricks table**
7. Select warehouse → catalog → schema → table

**While doing this, write down:**
- Which dropdown is populated by the **SDK** (control plane)?
- Which step uses the **SQL Connector** (data plane)?
- What identity is used locally vs what would be used when deployed as an App?

### Part B — Deployed (optional, 45 min)

1. Load repo as a Databricks Git folder
2. Create an App, deploy `streamlit/`
3. Grant the app's service principal `CAN USE` on a warehouse and `SELECT` on a test table
4. Repeat the same recipe in the deployed app
5. Open **Authentication → Get current user** and compare headers locally vs deployed

### Part C — Reflect (10 min)

Sketch on paper (or in this journal) your own team's equivalent: *Where would our DS team's app sit in Pattern A, B, or C?*

---

## 1.8 Module 1 journal entry

### Key concepts learned

- [ ] Databricks = governed storage (Delta/UC) + elastic compute + APIs
- [ ] Control plane (metadata, auth, scheduling) vs data plane (storage, warehouses, clusters)
- [ ] Three integration patterns: SQL Connector, SDK, Databricks Connect
- [ ] Apps identity model: service principal vs on-behalf-of-user
- [ ] This repo is the **consumption/application layer**, not the ingestion/processing layer
- [ ] `app.yaml` binds platform resources to application code at deploy time

### Important Databricks services (Module 1)

| Service | Introduced in this module |
|---------|---------------------------|
| Workspace | Container for all resources |
| Unity Catalog | Governance layer (detailed in M3) |
| SQL Warehouse | App query path |
| Clusters / Serverless | Spark execution path |
| Databricks Apps | App hosting runtime |
| Service principals & OBO | App authorization model |
| Databricks SDK | Control-plane API client |
| Lakebase | Mentioned; deep dive in M3/M6 |

### Architecture diagram (Module 1 — enterprise view)

```
                    ┌─────────────────────────────────────┐
                    │         Enterprise Users             │
                    │  (analysts, scientists, customers)   │
                    └──────────────┬──────────────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         ▼                         ▼                         ▼
   Databricks Apps            SQL / BI                 Notebooks/Jobs
   (this cookbook)            (Dashboards)             (ETL, training)
         │                         │                         │
         └─────────────────────────┼─────────────────────────┘
                                   ▼
                    ┌──────────────────────────────┐
                    │        Unity Catalog          │
                    │  tables · volumes · models    │
                    └──────────────┬───────────────┘
                                   ▼
                    ┌──────────────────────────────┐
                    │   Delta Lake (object storage) │
                    └──────────────────────────────┘
```

### Practical takeaways

1. **Before writing app code, map the request path:** user → app → which compute → which UC object → which identity.
2. **Choose integration pattern by workload:** SQL for reads, SDK for operations, Spark for transforms.
3. **OBO is a product feature, not a code hack** — the proxy injects tokens; your app decides whether to use them.
4. **The cookbook recipes are copy-paste building blocks**, not a system architecture — your medallion/jobs layer sits underneath.

### Questions answered correctly

*(Fill in after checkpoint — see below)*

| Question | Your answer | Correct? |
|----------|-------------|----------|
| Q1 | | |
| Q2 | | |
| Q3 | | |
| Q4 | | |

### Knowledge gaps to revisit

*(Fill in after checkpoint)*

- 

---

## 1.9 Checkpoint — answer before Module 2

Reply with your answers (short prose is fine). I will update the journal with what you got right and clarify gaps before we continue.

**Q1.** In `tables_read.py`, the warehouse dropdown is populated via `WorkspaceClient`. The table data is fetched via the SQL Connector. *Why are two different clients used instead of one?*

**Q2.** You deploy the Streamlit cookbook as a Databricks App. A user who lacks `SELECT` on a table opens the OBO recipe and runs a query. What happens, and why is that the correct behavior?

**Q3.** Your team wants an internal app for 50 data scientists to explore experiment results in a Gold table. Another team wants a customer-facing RAG chatbot over sensitive documents. Which auth mode (service principal vs OBO) would you recommend for each, and why?

**Q4.** This repository has almost no ingestion code (no Auto Loader, no DLT pipelines). *Where in a real enterprise architecture does ingestion live relative to these Apps recipes?* Sketch the boundary in one or two sentences.

---

*Next module (after checkpoint): **Module 2 — Data Ingestion** (batch/streaming patterns, how data enters the platform, and where `workflows_run` fits as the app's touchpoint to orchestration).*
