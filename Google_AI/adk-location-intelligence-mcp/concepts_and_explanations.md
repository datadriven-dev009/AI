# Build a Location Intelligence ADK Agent with MCP Servers
> **Codelab:** BigQuery + Google Maps via MCP | **Duration:** 45 mins | **Level:** All levels

---

## 1. What This Codelab Builds

An intelligent **Location Intelligence Agent** using:
- **ADK (Agent Development Kit)** — to build and orchestrate the agent
- **Gemini 3.1 Pro** — as the underlying LLM
- **BigQuery MCP Server** — to query demographic, pricing, and sales data
- **Google Maps MCP Server** — for real-world location analysis and validation
- A fictitious **bakery dataset** as the business use case

### Architecture Overview

```
User Query
    │
    ▼
LlmAgent (Gemini 3.1 Pro Preview)
    │
    ├──► BigQuery MCP Server ──► Demographics, Pricing, Sales, Foot Traffic
    │
    └──► Google Maps MCP Server ──► Places Search, Distance, Route Analysis
```

---

## 2. Key Concepts

### 2.1 MCP — Model Context Protocol
- A **standardized protocol** for AI agents to connect to external tools/services
- Eliminates the need to write complex API wrappers manually
- Google hosts **remote MCP servers** for BigQuery and Maps
- Agents connect via `StreamableHTTPConnectionParams` (HTTP-based, not local subprocess)

#### Two Types of MCP Connections:
| Type | How | Used Here |
|------|-----|-----------|
| **Local (stdio)** | Spawns a local process | Not used |
| **Remote (Streamable HTTP)** | HTTP connection to hosted server | ✅ Used |

---

### 2.2 MCPToolset
- ADK class that wraps an MCP server connection into **usable agent tools**
- The agent automatically discovers all tools exposed by the MCP server
- No need to manually define each tool — MCP introspection handles it

```python
tools = MCPToolset(
    connection_params=StreamableHTTPConnectionParams(
        url=MCP_SERVER_URL,
        headers={"Authorization": f"Bearer {token}"}
    )
)
```

---

### 2.3 Authentication Strategies

#### Maps MCP — API Key Auth
```
Maps MCP Server
    ← Header: X-Goog-Api-Key: <YOUR_MAPS_API_KEY>
```
- Simple static API key passed in request headers

#### BigQuery MCP — OAuth 2.0 Bearer Token
```
BigQuery MCP Server
    ← Header: Authorization: Bearer <OAUTH_TOKEN>
    ← Header: x-goog-user-project: <PROJECT_ID>
```
- Uses `google.auth.default()` to get Application Default Credentials (ADC)
- Token is refreshed before use via `credentials.refresh()`
- ⚠️ **Tokens expire after 60 mins** — re-run `gcloud auth application-default login` if session is long

---

### 2.4 LlmAgent
- Core ADK class for building LLM-powered agents
- Takes: `model`, `name`, `instruction`, `tools`
- The **system instruction** guides how the agent uses its tools
- Both toolsets (BigQuery + Maps) are passed as a flat list

---

### 2.5 BigQuery Dataset — `mcp_bakery`

| Table | Contents |
|-------|----------|
| `demographics` | Census data + population by zip code |
| `bakery_prices` | Competitor pricing and product details |
| `sales_history_weekly` | Weekly sales by store and product |
| `foot_traffic` | Foot traffic scores by zip code and time of day |

---

### 2.6 Agent Reasoning Flow (Example)

**User asks:** *"Find the best zip code in LA for a new bakery"*

```
Agent thinks:
  1. Query BigQuery foot_traffic → find zip with highest morning score
  2. Query BigQuery bakery_prices → check competitor pricing
  3. Call Google Maps Places Search → check bakery saturation in that zip
  4. Combine insights → give unified recommendation
```

This **unified multi-source reasoning** is the core value of the agent pattern.

---

### 2.7 Project Structure

```
launchmybakery/
├── data/                        # Pre-generated CSV files for BigQuery
├── adk_agent/                   # AI Agent Application
│   └── mcp_bakery_app/
│       ├── agent.py             # Agent definition (LlmAgent setup)
│       ├── tools.py             # MCP Toolset initialization
│       └── .env                 # Project config + API keys (auto-created)
├── setup/                       # Infrastructure setup scripts
│   ├── setup_env.sh             # Enables APIs, creates .env file
│   └── setup_bigquery.sh        # Creates GCS bucket + BQ dataset/tables
└── cleanup/
    └── cleanup_env.sh           # Deletes all created resources
```

---

### 2.8 ADK Web Interface
- `adk web` command spins up a **lightweight local web server** (port 8000)
- Provides a chat UI to interact with the agent
- In Cloud Shell, use `--allow_origins` flag for CORS with the Cloud Shell preview URL
- Press `Ctrl+C` to stop the server

---

### 2.9 What You Accomplished (Summary)

| Skill | What You Did |
|-------|-------------|
| **Infrastructure as Code** | Provisioned BigQuery dataset via CLI scripts |
| **MCP Integration** | Connected agent to 2 remote MCP servers without custom API wrappers |
| **Unified Reasoning** | Single agent combines BigQuery data + Maps location context |
| **Business Intelligence** | Solved real-world bakery site selection problems |

---

### 2.10 Key Terms Glossary

| Term | Meaning |
|------|---------|
| **ADK** | Agent Development Kit — Google's framework for building AI agents |
| **MCP** | Model Context Protocol — standard for agent-tool communication |
| **MCPToolset** | ADK class that connects to an MCP server |
| **LlmAgent** | ADK class for LLM-powered agents |
| **StreamableHTTPConnectionParams** | MCP connection type for remote HTTP servers |
| **ADC** | Application Default Credentials — Google auth mechanism |
| **BigQuery** | Google's serverless data warehouse |
| **foot_traffic** | Estimated pedestrian activity score by area and time |
