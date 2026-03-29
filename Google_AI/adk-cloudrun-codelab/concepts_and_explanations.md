# 🧠 ADK Agent on Cloud Run — Concepts & Explanations

---

## 1. Enabled APIs — What They Do

| **API** | **Service ID** | **Role in This Project** |
|---|---|---|
| **Cloud Run Admin API** | `run.googleapis.com` | Runs containerized services in a fully managed, auto-scalable environment. Hosts the ADK agent as a web service. |
| **Artifact Registry API** | `artifactregistry.googleapis.com` | Stores the Docker container image built from your code. Acts as the private container image repository. |
| **Cloud Build API** | `cloudbuild.googleapis.com` | Serverless CI/CD platform that builds your container image in the cloud from source code. |
| **Vertex AI API** | `aiplatform.googleapis.com` | Enables the agent to call Google's Gemini models. Unified API for all Google Cloud AI services. |
| **Compute Engine API** | `compute.googleapis.com` | Required as a foundational dependency for networking/compute resources even in managed environments like Cloud Run. |

---

## 2. Project Architecture Overview

```
User Request
     │
     ▼
┌──────────────────────────────────────┐
│         root_agent  (greeter)        │  ← Entry point; greets user,
│  saves prompt to state, delegates   │    calls add_prompt_to_state tool
└──────────────────┬───────────────────┘
                   │ transfers to
                   ▼
┌──────────────────────────────────────┐
│       tour_guide_workflow            │  ← SequentialAgent;
│       (SequentialAgent)              │    runs sub-agents in fixed order
└──────┬───────────────────────┬───────┘
       │ Step 1                │ Step 2
       ▼                       ▼
┌─────────────────┐   ┌──────────────────────┐
│ comprehensive   │   │  response_formatter  │
│  _researcher   │   │                      │
│                 │   │  Takes research_data │
│ Reads PROMPT,  │   │  from state and      │
│ queries         │   │  formats it into a  │
│ Wikipedia Tool  │   │  friendly response  │
│ → research_data │   │  for the user       │
└─────────────────┘   └──────────────────────┘
```

---

## 3. Key Concepts

### 🔷 Google ADK (Agent Development Kit)
The **Google ADK** is a framework for building AI agents powered by Gemini models. It provides standardized building blocks for:
- Defining agent behavior via natural language `instruction` prompts
- Connecting agents to **tools** (Python functions, APIs, LangChain tools)
- Orchestrating **multi-agent workflows**

---

### 🔷 Agent Types Used

| **Agent Type** | **Class** | **Behavior** |
|---|---|---|
| Standard Agent | `Agent` | Uses an LLM to reason and decide which tools or sub-agents to call |
| Sequential Agent | `SequentialAgent` | Runs a fixed list of sub-agents **in order** — no reasoning of its own |

> **Why use `SequentialAgent`?**
> When you need a **predictable, step-by-step pipeline** (e.g., always research first, then format), `SequentialAgent` removes LLM decision-making from orchestration, making the flow deterministic and reliable.

---

### 🔷 Agent State — Shared Memory (`tool_context.state`)

The **state dictionary** is the agent's short-term memory for a single conversation. Data written by one agent is available to all subsequent agents in the workflow.

**How data flows through state in this project:**

```
root_agent
  └─ writes → state["PROMPT"]  (via add_prompt_to_state tool)

comprehensive_researcher
  └─ reads  → { PROMPT }       (from state)
  └─ writes → state["research_data"]  (via output_key="research_data")

response_formatter
  └─ reads  → { research_data } (from state)
  └─ returns final response to user
```

> **`output_key`** on an agent automatically saves that agent's final text response into `state[output_key]`.

---

### 🔷 Tools Explained

#### ✅ `add_prompt_to_state` — Custom Python Function Tool
- A plain Python function used as an ADK tool by `root_agent`.
- **Purpose:** Saves the user's message into `state["PROMPT"]` so downstream agents can access it.
- **Why needed?** The `root_agent` handles the first turn of conversation. Saving the prompt explicitly makes it available to the `SequentialAgent` pipeline that runs next.

#### ✅ `LangchainTool` — Wikipedia Tool
- An **adapter class** that wraps any LangChain tool for use inside an ADK agent.
- Uses `WikipediaQueryRun` from `langchain-community` under the hood.
- **Purpose:** Gives the `comprehensive_researcher` access to factual, real-world Wikipedia knowledge (e.g., animal diets, habitats, lifespan).

---

### 🔷 The Three Agents — Roles Explained

| **Agent** | **Role** | **Input** | **Output** |
|---|---|---|---|
| `root_agent` (greeter) | Entry point; greets user, saves prompt, delegates | User message | Transfers to workflow |
| `comprehensive_researcher` | Fetches data using Wikipedia tool | `{ PROMPT }` from state | `research_data` stored in state |
| `response_formatter` | Formats raw data into friendly response | `{ research_data }` from state | Final answer to user |

---

## 4. Deployment Concepts

### 🔷 `uvx` — Run CLI Tools Without Installing
`uvx` runs a Python command-line tool **ephemerally** without globally installing it. This ensures you're using the exact pinned version of the ADK CLI:
```bash
uvx --from google-adk==1.14.0 adk deploy cloud_run ...
```

---

### 🔷 What `adk deploy cloud_run` Does Automatically

| **Step** | **What Happens Behind the Scenes** |
|---|---|
| 📦 Package code | Bundles your Python source files |
| 🔨 Build container | Uses **Cloud Build** to create a Docker image |
| 📤 Push to registry | Pushes the image to **Artifact Registry** |
| 🚀 Deploy service | Launches the container on **Cloud Run** |
| 🌐 Expose URL | Provides a public HTTPS endpoint for the agent |

---

### 🔷 `--with_ui` Flag
Includes a **built-in web chat interface** with the deployed service. Lets you test the agent directly in a browser — no custom frontend needed.

---

### 🔷 Service Account & Least Privilege Security

A **dedicated service account** (`lab2-cr-service`) is created specifically for this Cloud Run service.

- ✅ Follows the **principle of least privilege** — the service only gets `roles/aiplatform.user`, not broad default permissions.
- ✅ Isolates this app's identity from other services in the project.
- ✅ Best practice for production security on Google Cloud.

---

## 5. Environment & Dependency Management

### 🔷 `uv` — Fast Python Package Manager
`uv` is a modern, fast drop-in replacement for `pip` and `venv`:

| **Command** | **Purpose** |
|---|---|
| `uv venv` | Creates an isolated virtual environment |
| `source .venv/bin/activate` | Activates the virtual environment |
| `uv pip install -r requirements.txt` | Installs all packages inside the venv |

> ⚠️ If Cloud Shell refreshes, re-run `source .venv/bin/activate` to reactivate.

---

### 🔷 `.env` File & `load_dotenv()`
Sensitive configuration (Project ID, model name, service account) is stored in a `.env` file and loaded at runtime using `load_dotenv()`.

**Benefits:**
- Keeps credentials **out of source code**
- Easy to change config without modifying `agent.py`
- Works seamlessly across local dev and Cloud Shell

---

## 6. Clean Up — Why It Matters

Undeleted resources continue to **incur charges** on your Google Cloud account.

| **Resource** | **Charge Risk** |
|---|---|
| Cloud Run Service | Billed per request + minimum instances |
| Artifact Registry images | Billed by storage size |
| Entire Project | All resources inside billed |

> ⚠️ **WARNING:** `gcloud projects delete` is **irreversible**. Only use if you created the project solely for this lab.
