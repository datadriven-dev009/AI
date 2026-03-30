# MCP Toolbox for Databases — Concepts & Explanations
> **Codelab:** Making BigQuery datasets available to MCP clients via ADK | **Duration:** 90 mins

---

## 1. What This Codelab Builds

A natural language agent that:
- Connects to a **public BigQuery dataset** (Google Cloud Release Notes)
- Exposes it via **MCP Toolbox for Databases** as an MCP server
- Uses an **ADK Agent** (Gemini 2.5 Flash) to answer user queries about release notes in plain English

### Architecture Overview

```
User Query (natural language)
        │
        ▼
  ADK Agent (Gemini 2.5 Flash)
        │
        ▼  HTTP (port 5000)
MCP Toolbox for Databases  ←── tools.yaml config
        │
        ▼
  BigQuery Public Dataset
  (google_cloud_release_notes)
```

---

## 2. Key Concepts

### 2.1 MCP Toolbox for Databases

An **open-source MCP server** built specifically for databases. Unlike building raw API wrappers, Toolbox handles:

| Feature | What It Does |
|---------|-------------|
| **Connection Pooling** | Reuses DB connections efficiently |
| **Authentication** | Handles credentials and auth flows |
| **Observability** | Built-in OpenTelemetry metrics + tracing |
| **Centralized Config** | One `tools.yaml` controls everything |
| **MCP Compatibility** | Automatically exposes tools as MCP endpoints |

> Think of it as a **middleware layer** between your AI agents and your databases.

### Supported Databases
AlloyDB, BigQuery, Cloud SQL (PostgreSQL/MySQL), Spanner, and more.

---

### 2.2 tools.yaml — The Core Config File

The `tools.yaml` file is the single configuration point for Toolbox. It has **three sections**:

#### Sources
Defines **where** to connect (your database):
```yaml
sources:
  my-bq-source:
    kind: bigquery
    project: YOUR_PROJECT_ID
```
- `kind` = database type (`bigquery`, `alloydb`, etc.)
- Each source gets a unique name used by tools

#### Tools
Defines **what** the agent can do (SQL queries):
```yaml
tools:
  search_release_notes_bq:
    kind: bigquery-sql
    source: my-bq-source
    statement: |
      SELECT product_name, description, published_at
      FROM `bigquery-public-data`.`google_cloud_release_notes`.`release_notes`
      WHERE DATE(published_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
      ORDER BY published_at DESC
    description: |
      Use this tool to get information on Google Cloud Release Notes.
```
- `description` is critical — the LLM reads this to decide **when to call the tool**
- `statement` is the actual SQL that runs against BigQuery

#### Toolsets
Groups tools together for different agents/apps:
```yaml
toolsets:
  my_bq_toolset:
    - search_release_notes_bq
```
- An agent loads a **toolset** (not individual tools)
- Supports multiple toolsets for different use cases

---

### 2.3 How Toolbox Compares to Previous Approach

| Aspect | Previous (Location Intel) | This Codelab (MCP Toolbox) |
|--------|--------------------------|---------------------------|
| MCP Server | Google-hosted remote server | **Self-hosted Toolbox binary** |
| Config | Code in `tools.py` | Declarative `tools.yaml` |
| Auth | OAuth token in headers | ADC via `gcloud auth` |
| SQL Control | Google-managed | **You write the SQL** |
| Customization | Limited | **Fully customizable** |

---

### 2.4 toolbox-core Python Package

The ADK agent connects to the Toolbox server using the `toolbox-core` package:

```python
from toolbox_core import ToolboxSyncClient

toolbox = ToolboxSyncClient("http://127.0.0.1:5000")
tools = toolbox.load_toolset('my_bq_toolset')
```

- `ToolboxSyncClient` — synchronous HTTP client to the Toolbox server
- `load_toolset()` — fetches all tools in the named toolset from the running server
- The agent then uses these tools just like any other ADK tool

---

### 2.5 Google Cloud Public Datasets

A **free, publicly queryable** program offering curated datasets in BigQuery:
- No data ingestion needed — datasets are already hosted
- Referenced as `bigquery-public-data.<dataset>.<table>`
- Examples: COVID-19 data, GitHub repos, NOAA weather, **Google Cloud Release Notes**

The Release Notes dataset mirrors the official [Google Cloud Release Notes](https://cloud.google.com/release-notes) page and is updated continuously.

---

### 2.6 ADK Agent Structure (Scaffolded by `adk create`)

```
gcp_releasenotes_agent_app/
├── .env           # GOOGLE_GENAI_USE_VERTEXAI, PROJECT_ID, REGION
├── __init__.py    # imports agent from agent.py
└── agent.py       # LlmAgent definition
```

- `adk create <app_name>` — scaffolds this structure automatically
- `.env` is auto-populated from your gcloud config
- `agent.py` starts with a minimal agent you then extend

---

### 2.7 ADK Agent vs Tool vs Toolset

| Term | ADK Meaning |
|------|-------------|
| **Agent** | Self-contained AI unit that reasons, plans, and acts |
| **LlmAgent / Agent** | Agent powered by an LLM (Gemini); decides which tools to call |
| **Tool** | A specific capability (function, API call, DB query) the agent can invoke |
| **Toolset** | A group of related tools loaded together from MCP Toolbox |

---

### 2.8 Toolbox API Endpoints

When the Toolbox server is running, it exposes:

| Endpoint | What It Returns |
|----------|----------------|
| `/api/toolset` | JSON list of all configured tools + descriptions |
| `/ui` | Web UI to browse and test tools interactively |
| `/` | Server version info |

---

### 2.9 MCP Toolbox vs Remote MCP Server (Comparison)

| | MCP Toolbox for Databases | Google Remote MCP (prev codelab) |
|--|--|--|
| **Hosting** | You run it locally / on your server | Google hosts it |
| **SQL** | You define custom SQL | Pre-built tools only |
| **Dataset** | Any BigQuery dataset / DB you own | Public Google APIs |
| **Auth** | ADC (Application Default Credentials) | OAuth / API Key headers |
| **Flexibility** | High — fully configurable | Low — fixed tool set |
| **Use Case** | Your own data, private datasets | Google Maps, BigQuery APIs |

---

### 2.10 Toolbox Server Startup — What the Logs Mean

```
INFO "Initialized 1 sources: my-bq-source"       ← DB connection ready
INFO "Initialized 1 tools: search_release_notes_bq"  ← SQL tool registered
INFO "Initialized 2 toolsets: my_bq_toolset, default" ← default auto-created
INFO "Server ready to serve!"                     ← HTTP server on port 5000
INFO "Toolbox UI is up and running at: http://127.0.0.1:5000/ui"  ← UI mode
WARN "wildcard (*) allows all origins..."         ← use --allowed-origins in prod
```

- A **`default` toolset** is always auto-created containing all defined tools
- The `--ui` flag enables the visual testing interface

---

### 2.11 Key Terms Glossary

| Term | Meaning |
|------|---------|
| **MCP Toolbox** | Open-source self-hosted MCP server for databases |
| **tools.yaml** | Declarative config file defining sources, tools, and toolsets |
| **Source** | A database connection config in tools.yaml |
| **Tool** | A SQL query or action exposed to the agent |
| **Toolset** | A named group of tools |
| **toolbox-core** | Python package to connect ADK agent to Toolbox |
| **ToolboxSyncClient** | Sync HTTP client class from toolbox-core |
| **ADC** | Application Default Credentials — Google auth via `gcloud auth` |
| **adk create** | CLI command to scaffold a new ADK agent app |
| **adk run** | CLI command to run agent in interactive terminal mode |
| **adk web** | CLI command to run agent with web chat UI |
