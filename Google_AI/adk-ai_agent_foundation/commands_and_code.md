# ADK AI Agent Foundation — Commands & Code

---

## 1. Configure Google Cloud

```bash
# Set your project ID
gcloud config set project <your-project-id>

# Enable Vertex AI API
gcloud services enable aiplatform.googleapis.com
```

> 🔗 Create project: [console.cloud.google.com/projectcreate](https://console.cloud.google.com/projectcreate)  
> 🔗 Launch Cloud Shell: [shell.cloud.google.com](https://shell.cloud.google.com)

---

## 2. Python Virtual Environment

```bash
# Create project folder
mkdir ai-agents-adk
cd ai-agents-adk

# Create & activate venv using uv (fast Rust-based tool)
uv venv --python 3.12
source .venv/bin/activate        # Mac/Linux
# .venv\Scripts\activate         # Windows

# Install ADK
uv pip install google-adk
```

> ⚠️ If terminal closes, re-run: `source .venv/bin/activate`

---

## 3. Create the Agent

```bash
adk create personal_assistant
```

**When prompted, select:**
| Step | Choice |
|------|--------|
| Model | `1` → gemini-2.5-flash |
| Backend | `2` → Vertex AI |
| Project ID | Press Enter (or type your ID) |
| Region | Press Enter → `us-central1` |

**Files created:**
```
personal_assistant/
├── agent.py       ← Agent logic
├── __init__.py    ← Package marker
└── .env           ← Config & credentials
```

---

## 4. Agent File Contents

**`agent.py`**
```python
from google.adk.agents import Agent

root_agent = Agent(
    model='gemini-2.5-flash',
    name='root_agent',
    description='A helpful assistant for user questions.',
    instruction='Answer user questions to the best of your knowledge',
)
```

**`__init__.py`**
```python
from . import agent
```

**`.env`**
```env
GOOGLE_GENAI_USE_VERTEXAI=1
GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
GOOGLE_CLOUD_LOCATION=YOUR_PROJECT_LOCATION
```

---

## 5. Run the Agent

**Terminal mode:**
```bash
adk run personal_assistant
# Type 'exit' to quit
```

**Web UI mode:**
```bash
adk web
# Open: http://localhost:8000
```

---

## 6. Clean Up (Optional)

```bash
# Delete project files
cd ~
rm -rf ai-agents-adk

# Disable Vertex AI API
gcloud services disable aiplatform.googleapis.com
```

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `billing to be enabled` | Check Project ID in `.env`, link billing account |
| `Vertex AI API has not been used` | Run `gcloud services enable aiplatform.googleapis.com` |
| Other errors | Reload Cloud Shell tab in browser |
