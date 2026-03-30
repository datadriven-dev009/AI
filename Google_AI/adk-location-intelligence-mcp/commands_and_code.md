# Commands & Code — Location Intelligence ADK Agent (MCP)
> **Codelab:** BigQuery + Google Maps via MCP Servers

---

## 1. Before You Begin — GCP Project Setup

```bash
# Verify authentication
gcloud auth list

# Confirm active project
gcloud config get project

# Set project ID as env variable
export PROJECT_ID=$(gcloud config get project)
```

---

## 2. Get the Code

```bash
# Clone the MCP repository
git clone https://github.com/google/mcp.git

# Navigate to the demo directory
cd mcp/examples/launchmybakery

# Authenticate for ADK to access BigQuery (Application Default Credentials)
gcloud auth application-default login
```

> ⚠️ ADK does NOT auto-refresh OAuth tokens. Re-run `gcloud auth application-default login` if session exceeds 60 minutes.

---

## 3. Configure Environment and BigQuery

```bash
# Make setup scripts executable
chmod +x setup/setup_env.sh
chmod +x ./setup/setup_bigquery.sh

# Run environment setup (enables APIs, creates .env with PROJECT_ID + MAPS_API_KEY)
./setup/setup_env.sh

# Run BigQuery setup (creates GCS bucket, uploads data, provisions dataset + tables)
./setup/setup_bigquery.sh
```

### BigQuery Tables Created:
| Table | Description |
|-------|-------------|
| `demographics` | Census data by zip code |
| `bakery_prices` | Competitor pricing + products |
| `sales_history_weekly` | Weekly sales by store/product |
| `foot_traffic` | Foot traffic scores by zip + time of day |

---

## 4. Install ADK

```bash
# Create virtual Python environment
python3 -m venv .venv

# Activate virtual environment
source .venv/bin/activate

# Install ADK
pip install google-adk==1.28.0

# Navigate to agent directory
cd adk_agent/
```

---

## 5. Key Code — tools.py

### MCP Server URLs
```python
MAPS_MCP_URL = "https://maps.googleapis.com/maps/api/mcp"       # Google Maps MCP
BIGQUERY_MCP_URL = "https://bigquery.googleapis.com/mcp"         # BigQuery MCP
```

### Maps MCP Toolset (API Key Auth)
```python
import os
import dotenv
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset
from google.adk.tools.mcp_tool.mcp_toolset import StreamableHTTPConnectionParams

def get_maps_mcp_toolset():
    dotenv.load_dotenv()
    maps_api_key = os.getenv('MAPS_API_KEY', 'no_api_found')
    
    tools = MCPToolset(
        connection_params=StreamableHTTPConnectionParams(
            url=MAPS_MCP_URL,
            headers={    
                "X-Goog-Api-Key": maps_api_key
            }
        )
    )
    print("MCP Toolset configured for Streamable HTTP connection.")
    return tools
```

### BigQuery MCP Toolset (OAuth 2.0 Auth)
```python
import google.auth
import google.auth.transport.requests

def get_bigquery_mcp_toolset():   
    credentials, project_id = google.auth.default(
        scopes=["https://www.googleapis.com/auth/bigquery"]
    )

    credentials.refresh(google.auth.transport.requests.Request())
    oauth_token = credentials.token
        
    HEADERS_WITH_OAUTH = {
        "Authorization": f"Bearer {oauth_token}",
        "x-goog-user-project": project_id
    }

    tools = MCPToolset(
        connection_params=StreamableHTTPConnectionParams(
            url=BIGQUERY_MCP_URL,
            headers=HEADERS_WITH_OAUTH
        )
    )
    print("MCP Toolset configured for Streamable HTTP connection.")
    return tools
```

---

## 6. Key Code — agent.py

```python
from google.adk.agents import LlmAgent
import tools

maps_toolset = tools.get_maps_mcp_toolset()
bigquery_toolset = tools.get_bigquery_mcp_toolset()

project_id = os.getenv('PROJECT_ID')

root_agent = LlmAgent(
    model='gemini-3.1-pro-preview',
    name='root_agent',
    instruction=f"""
        Help the user answer questions by strategically combining insights from two sources:
        
        1. **BigQuery toolset:** Access demographic (inc. foot traffic index), product pricing,
           and historical sales data in the mcp_bakery dataset. Do not use any other dataset.
           Run all query jobs from project id: {project_id}.

        2. **Maps Toolset:** Use this for real-world location analysis, finding competition/places
           and calculating necessary travel routes.
           Include a hyperlink to an interactive map in your response where appropriate.
    """,
    tools=[maps_toolset, bigquery_toolset]
)
```

---

## 7. Run the Agent (ADK Web UI)

```bash
# Make sure you're in adk_agent/ directory
cd adk_agent/

# Start the ADK web interface
adk web --allow_origins 'regex:https://.*\.cloudshell\.dev'
```

Access the UI:
- **Option 1:** Click `http://127.0.0.1:8000` in terminal
- **Option 2:** Cloud Shell → Web Preview → Change port → `8000`

Stop the server: `Ctrl+C`

---

## 8. Sample Agent Prompts

```
# 1. Find best neighborhood
"I want to open a bakery in Los Angeles. Find the zip code with the highest morning foot traffic score."

# 2. Check market saturation
"Can you search for 'Bakeries' in that zip code to see if it's saturated?"

# 3. Advanced combo query
"I'm looking to open my fourth bakery location in Los Angeles. I need a neighborhood with early activity.
Find the zip code with the highest 'morning' foot traffic score."

# 4. Check competition + alternatives
"Can you search for 'Bakeries' in that zip code to see if it's saturated? If there are too many,
check for 'Specialty Coffee' shops, so I can position myself near them to capture foot traffic."

# 5. Premium pricing
"Okay and I want to position this as a premium brand. What is the maximum price being charged
for a 'Sourdough Loaf' in the LA Metro area?"

# 6. Revenue projection
"Now I want a revenue projection for December 2025. Look at my sales history and take data from
my best performing store for the 'Sourdough Loaf'. Run a forecast for December 2025 to estimate
the quantity I'll sell. Then calculate the projected total revenue using just under the premium
price we found (let's use $18)"

# 7. Logistics check
"That'll cover my rent. Lastly, let's verify logistics. Find the closest 'Restaurant Depot'
to the proposed area and make sure that drive time is under 30 minutes for daily restocking."
```

---

## 9. Clean Up

```bash
# Make cleanup script executable + run it
chmod +x ../cleanup/cleanup_env.sh
./../cleanup/cleanup_env.sh
```

This deletes:
- BigQuery dataset (`mcp_bakery`)
- Cloud Storage bucket
- Created API keys

---

## 10. Reference Links

| Resource | URL |
|----------|-----|
| BigQuery MCP Server docs | https://cloud.google.com/bigquery/docs/mcp |
| Maps Grounding Lite MCP | https://developers.google.com/maps/documentation/mcp |
| Authenticate MCP | https://cloud.google.com/vertex-ai/docs/mcp-auth |
| ADK Documentation | https://google.github.io/adk-docs/ |
| GitHub Repo | https://github.com/google/mcp |
