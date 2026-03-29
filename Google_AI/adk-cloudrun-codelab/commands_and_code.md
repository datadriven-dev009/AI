# 🚀 ADK Agent on Cloud Run — Commands & Code Reference

---

## 1. Project Setup

### Set Active Project
```bash
gcloud config set project [PROJECT_ID]
# Output: Updated property [core/project].
```

### Enable Required APIs
```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  aiplatform.googleapis.com \
  compute.googleapis.com
# Output: Operation "operations/acat.p2-[GUID]" finished successfully.
```

---

## 2. Create Project Directory

```bash
cd && mkdir zoo_guide_agent && cd zoo_guide_agent
cloudshell open-workspace ~/zoo_guide_agent
```

---

## 3. Install Requirements

### Create `requirements.txt`
```bash
cloudshell edit requirements.txt
```

**Paste into `requirements.txt`:**
```
google-adk==1.14.0
langchain-community==0.3.27
wikipedia==1.4.0
```

### Create & Activate Virtual Environment
```bash
uv venv
source .venv/bin/activate
```

> ⚠️ **Note:** If Cloud Shell refreshes, re-run `source .venv/bin/activate` to reactivate.

### Install Packages
```bash
uv pip install -r requirements.txt
```

---

## 4. Set Up Environment Variables

```bash
# Step 1: Set variables in terminal
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
SA_NAME=lab2-cr-service

# Step 2: Write to .env file
cat <<EOF > .env
PROJECT_ID=$PROJECT_ID
PROJECT_NUMBER=$PROJECT_NUMBER
SA_NAME=$SA_NAME
SERVICE_ACCOUNT=${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com
MODEL="gemini-2.5-flash"
EOF
```

> **Useful project info commands:**
> ```bash
> gcloud projects list
> gcloud iam service-accounts list
> ```

---

## 5. Create Agent Files

### `__init__.py`
```bash
cloudshell edit __init__.py
```

```python
from . import agent
```

---

### `agent.py`
```bash
cloudshell edit agent.py
```

```python
import os
import logging
import google.cloud.logging
from dotenv import load_dotenv

from google.adk import Agent
from google.adk.agents import SequentialAgent
from google.adk.tools.tool_context import ToolContext
from google.adk.tools.langchain_tool import LangchainTool

from langchain_community.tools import WikipediaQueryRun
from langchain_community.utilities import WikipediaAPIWrapper

import google.auth
import google.auth.transport.requests
import google.oauth2.id_token

# --- Setup Logging and Environment ---
cloud_logging_client = google.cloud.logging.Client()
cloud_logging_client.setup_logging()

load_dotenv()

model_name = os.getenv("MODEL")

# ── Tool 1: Save user prompt to state ──────────────────────────────────────
def add_prompt_to_state(
    tool_context: ToolContext, prompt: str
) -> dict[str, str]:
    """Saves the user's initial prompt to the state."""
    tool_context.state["PROMPT"] = prompt
    logging.info(f"[State updated] Added to PROMPT: {prompt}")
    return {"status": "success"}

# ── Tool 2: Wikipedia Tool via LangChain ───────────────────────────────────
wikipedia_tool = LangchainTool(
    tool=WikipediaQueryRun(api_wrapper=WikipediaAPIWrapper())
)

# ── Agent 1: Researcher ────────────────────────────────────────────────────
comprehensive_researcher = Agent(
    name="comprehensive_researcher",
    model=model_name,
    description="The primary researcher that can access both internal zoo data and external knowledge from Wikipedia.",
    instruction="""
    You are a helpful research assistant. Your goal is to fully answer the user's PROMPT.
    You have access to two tools:
    1. A tool for getting specific data about animals AT OUR ZOO (names, ages, locations).
    2. A tool for searching Wikipedia for general knowledge (facts, lifespan, diet, habitat).

    First, analyze the user's PROMPT.
    - If the prompt can be answered by only one tool, use that tool.
    - If the prompt is complex and requires information from both the zoo's database AND Wikipedia,
      you MUST use both tools to gather all necessary information.
    - Synthesize the results from the tool(s) you use into preliminary data outputs.

    PROMPT:
    { PROMPT }
    """,
    tools=[wikipedia_tool],
    output_key="research_data"
)

# ── Agent 2: Response Formatter ────────────────────────────────────────────
response_formatter = Agent(
    name="response_formatter",
    model=model_name,
    description="Synthesizes all information into a friendly, readable response.",
    instruction="""
    You are the friendly voice of the Zoo Tour Guide. Your task is to take the
    RESEARCH_DATA and present it to the user in a complete and helpful answer.

    - First, present the specific information from the zoo (like names, ages, and where to find them).
    - Then, add the interesting general facts from the research.
    - If some information is missing, just present the information you have.
    - Be conversational and engaging.

    RESEARCH_DATA:
    { research_data }
    """
)

# ── Workflow Agent: Sequential Pipeline ───────────────────────────────────
tour_guide_workflow = SequentialAgent(
    name="tour_guide_workflow",
    description="The main workflow for handling a user's request about an animal.",
    sub_agents=[
        comprehensive_researcher,  # Step 1: Gather all data
        response_formatter,        # Step 2: Format the final response
    ]
)

# ── Root Agent: Entry Point ────────────────────────────────────────────────
root_agent = Agent(
    name="greeter",
    model=model_name,
    description="The main entry point for the Zoo Tour Guide.",
    instruction="""
    - Let the user know you will help them learn about the animals we have in the zoo.
    - When the user responds, use the 'add_prompt_to_state' tool to save their response.
    After using the tool, transfer control to the 'tour_guide_workflow' agent.
    """,
    tools=[add_prompt_to_state],
    sub_agents=[tour_guide_workflow]
)
```

---

## 6. Expected Project Structure

```
zoo_guide_agent/
├── .env
├── __init__.py
├── agent.py
└── requirements.txt
```

---

## 7. IAM Permissions Setup

```bash
# Load environment variables
source .env

# Create a dedicated service account
gcloud iam service-accounts create ${SA_NAME} \
    --display-name="Service Account for lab 2"

# Grant Vertex AI User role
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SERVICE_ACCOUNT" \
  --role="roles/aiplatform.user"
```

---

## 8. Deploy the Agent

```bash
uvx --from google-adk==1.14.0 \
adk deploy cloud_run \
  --project=$PROJECT_ID \
  --region=europe-west1 \
  --service_name=zoo-tour-guide \
  --with_ui \
  . \
  -- \
  --labels=dev-tutorial=codelab-adk \
  --service-account=$SERVICE_ACCOUNT
```

> ⚠️ If prompted:
> - **"Do you want to continue (Y/n)?"** → Type `Y` and press Enter
> - **"Allow unauthenticated invocations? (y/N)"** → Type `y` and press Enter

✅ On success, a public URL will be shown:
```
https://zoo-tour-guide-XXXXXXXXX.europe-west1.run.app
```

---

## 9. Clean Up Resources

### Delete Cloud Run Service & Container Image
```bash
gcloud run services delete zoo-tour-guide --region=europe-west1 --quiet
gcloud artifacts repositories delete cloud-run-source-deploy --location=europe-west1 --quiet
```

### (Optional) Delete Entire Project
```bash
gcloud projects delete $PROJECT_ID
```

> ⚠️ **WARNING:** Deleting a project is **irreversible**. Ensure the project is not used elsewhere.
