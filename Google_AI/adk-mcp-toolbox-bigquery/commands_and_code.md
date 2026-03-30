# Commands & Code — MCP Toolbox for Databases (BigQuery + ADK)
> **Codelab:** Making BigQuery datasets available to MCP clients

---

## 1. Before You Begin — GCP Project Setup

```bash
# Verify authentication
gcloud auth list

# Verify active project
gcloud config list project

# Set project if needed
gcloud config set project <YOUR_PROJECT_ID>

# Enable required APIs
gcloud services enable cloudresourcemanager.googleapis.com \
                       servicenetworking.googleapis.com \
                       run.googleapis.com \
                       cloudbuild.googleapis.com \
                       cloudfunctions.googleapis.com \
                       aiplatform.googleapis.com \
                       sqladmin.googleapis.com \
                       compute.googleapis.com
```

---

## 2. BigQuery — Validate the Public Dataset

Run this in BigQuery Console to verify the public dataset works:

```sql
SELECT
  product_name,
  description,
  published_at
FROM
  `bigquery-public-data`.`google_cloud_release_notes`.`release_notes`
WHERE
  DATE(published_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
GROUP BY product_name, description, published_at
ORDER BY published_at DESC
```

---

## 3. Install MCP Toolbox for Databases

```bash
# Create and enter the toolbox folder
mkdir mcp-toolbox
cd mcp-toolbox

# Download the binary (Linux/Cloud Shell)
export VERSION=0.23.0
curl -O https://storage.googleapis.com/genai-toolbox/v$VERSION/linux/amd64/toolbox
chmod +x toolbox
```

> For Mac or Windows binaries, check the [releases page](https://github.com/googleapis/mcp-toolbox-for-databases/releases).

---

## 4. Configure tools.yaml

Create `tools.yaml` in the `mcp-toolbox` folder:

```bash
# Open nano editor to create the config file
nano tools.yaml
```

**Contents of tools.yaml** (replace `YOUR_PROJECT_ID`):

```yaml
sources:
  my-bq-source:
    kind: bigquery
    project: YOUR_PROJECT_ID

tools:
  search_release_notes_bq:
    kind: bigquery-sql
    source: my-bq-source
    statement: |
      SELECT
        product_name,
        description,
        published_at
      FROM
        `bigquery-public-data`.`google_cloud_release_notes`.`release_notes`
      WHERE
        DATE(published_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
      GROUP BY product_name, description, published_at
      ORDER BY published_at DESC
    description: |
      Use this tool to get information on Google Cloud Release Notes.

toolsets:
  my_bq_toolset:
    - search_release_notes_bq
```

---

## 5. Run and Test MCP Toolbox Server

```bash
# Start Toolbox server (default port 5000)
./toolbox --tools-file="tools.yaml"

# If port 5000 is busy, use another port
./toolbox --tools-file "tools.yaml" --port 7000

# Start with UI enabled (visual tool tester)
./toolbox --tools-file "tools.yaml" --ui
```

### Expected Startup Output:
```
INFO "Initialized 1 sources: my-bq-source"
INFO "Initialized 0 authServices: "
INFO "Initialized 1 tools: search_release_notes_bq"
INFO "Initialized 2 toolsets: my_bq_toolset, default"
INFO "Initialized 0 prompts: "
INFO "Initialized 1 promptsets: default"
INFO "Server ready to serve!"
# (with --ui flag):
INFO "Toolbox UI is up and running at: http://127.0.0.1:5000/ui"
```

### Verify via API:
```bash
# Check configured tools via API endpoint
curl http://127.0.0.1:5000/api/toolset
```

Expected JSON response:
```json
{
  "serverVersion": "0.22.0+binary.linux.amd64...",
  "tools": {
    "search_release_notes_bq": {
      "description": "Use this tool to get information on Google Cloud Release Notes.\n",
      "parameters": [],
      "authRequired": []
    }
  }
}
```

### Verify via UI:
- Open browser → `http://127.0.0.1:5000/ui`
- Click **Tools** on the left
- Click `search_release_notes_bq`
- Click **Run Tool** (no parameters needed)

---

## 6. Install ADK and Create Agent

```bash
# Open a NEW terminal tab — keep Toolbox running in the first tab

# Create and enter agents folder
mkdir my-agents
cd my-agents

# Create virtual Python environment
python -m venv .venv

# Activate virtual environment
source .venv/bin/activate          # Linux / Cloud Shell / Mac
# .venv\Scripts\activate           # Windows

# Install ADK + MCP Toolbox Python client
pip install google-adk toolbox-core

# Verify ADK installed
adk
```

### Scaffold Agent with ADK:
```bash
adk create gcp_releasenotes_agent_app
```

**Follow the interactive prompts:**
```
Choose model for root agent:
  1. gemini-2.5-flash         ← Select this
  2. Other models

Choose a backend:
  1. Google AI
  2. Vertex AI               ← Select this

Enter Google Cloud project ID [auto-detected]:   ← press Enter
Enter Google Cloud region [us-central1]:         ← press Enter
```

---

## 7. Generated File Structure

```
my-agents/
└── gcp_releasenotes_agent_app/
    ├── .env           # Auto-populated from gcloud config
    ├── __init__.py    # Module entry point
    └── agent.py       # Agent definition (edit this)
```

### .env (auto-generated):
```env
GOOGLE_GENAI_USE_VERTEXAI=1
GOOGLE_CLOUD_PROJECT=YOUR_GOOGLE_PROJECT_ID
GOOGLE_CLOUD_LOCATION=YOUR_GOOGLE_PROJECT_REGION
```

### __init__.py (auto-generated):
```python
from . import agent
```

### agent.py (default scaffold):
```python
from google.adk.agents import Agent

root_agent = Agent(
    model='gemini-2.5-flash',
    name='root_agent',
    description='A helpful assistant for user questions.',
    instruction='Answer user questions to the best of your knowledge',
)
```

---

## 8. Connect Agent to MCP Toolbox — Final agent.py

Replace the contents of `agent.py` with:

```python
from google.adk.agents import Agent
from toolbox_core import ToolboxSyncClient

# Connect to the running Toolbox server
toolbox = ToolboxSyncClient("http://127.0.0.1:5000")

# Load all tools from the named toolset
tools = toolbox.load_toolset('my_bq_toolset')

root_agent = Agent(
    name="gcp_releasenotes_agent",
    model="gemini-2.5-flash",
    description=(
        "Agent to answer questions about Google Cloud Release notes."
    ),
    instruction=(
        "You are a helpful agent who can answer user questions about the "
        "Google Cloud Release notes. Use the tools to answer the question."
    ),
    tools=tools,
)
```

> ⚠️ If using a custom port (e.g., 7000), change `5000` → `7000` in `ToolboxSyncClient`.

---

## 9. Run the Agent

```bash
# Terminal 1 — Toolbox must be running
./toolbox --tools-file="tools.yaml"

# Terminal 2 — from my-agents folder
adk run gcp_releasenotes_agent_app/

# Or with web UI:
adk web
```

### Sample Agent Session (adk run):
```
[user]: get me the google cloud release notes

[gcp_releasenotes_agent]: Here are the Google Cloud Release Notes.

Google SecOps SOAR: Release 6.3.49 is being rolled out...
Compute Engine: Dynamic NICs let you add or remove network interfaces...
Google Agentspace: App-level feature management...
...
```

---

## 10. Sample Prompts to Test the Agent

```
"Get me the latest Google Cloud release notes"
"What was released in Compute Engine recently?"
"Show me all Security Command Center updates from the past week"
"Are there any new Vertex AI features?"
"What Google Cloud products had releases today?"
```

---

## 11. Reference Links

| Resource | URL |
|----------|-----|
| MCP Toolbox for Databases GitHub | https://github.com/googleapis/mcp-toolbox-for-databases |
| Toolbox Releases (binaries) | https://github.com/googleapis/mcp-toolbox-for-databases/releases |
| BigQuery Source Config Docs | https://googleapis.github.io/mcp-toolbox-for-databases/sources/bigquery/ |
| BigQuery Public Datasets | https://cloud.google.com/bigquery/public-data |
| Google Cloud Release Notes Dataset | https://console.cloud.google.com/bigquery?p=bigquery-public-data&d=google_cloud_release_notes |
| ADK Documentation | https://google.github.io/adk-docs/ |
| toolbox-core PyPI | https://pypi.org/project/toolbox-core/ |
