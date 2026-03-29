# 🧠 ADK Agent on Cloud Run — Concepts & Explanations

> [!TIP]
> **TL;DR Highlights**
> - **ADK** = Google's framework for building agents with Gemini.
> - **State** = Shared memory dictionary (`PROMPT`, `research_data`) passed between agents.
> - **SequentialAgent** = Runs sub-agents in a fixed 1 → 2 → 3 order (deterministic).
> - **Agent** = Uses an LLM to "think" and decide.
> - **`adk deploy cloud_run`** = Automates building, pushing, and launching your agent.

---

## 1. Enabled APIs — What They Do

| **API** | **Service ID** | **Role in This Project** |
|---|---|---|
| **Cloud Run Admin API** | `run.googleapis.com` | **Runs** your containerized agent as a web service. |
| **Artifact Registry API** | `artifactregistry.googleapis.com` | **Stores** your container images (Docker). |
| **Cloud Build API** | `cloudbuild.googleapis.com` | **Builds** your container in the cloud from source. |
| **Vertex AI API** | `aiplatform.googleapis.com` | **Connects** your agent to the Gemini models. |
| **Compute Engine API** | `compute.googleapis.com` |的基础 dependency for networking resources. |

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

> [!IMPORTANT]
> **Choosing the right agent class:**
> - Use **`Agent`** when you want the AI to handle decisions and tools dynamically.
> - Use **`SequentialAgent`** when you want a fixed, reliable pipeline (e.g., always Research first, then Format).

| **Agent Type** | **Class** | **Behavior** |
|---|---|---|
| Standard Agent | `Agent` | Uses an LLM to reason and decide which tools or sub-agents to call |
| Sequential Agent | `SequentialAgent` | Runs a fixed list of sub-agents **in order** — no reasoning of its own |

---

### 🔷 Agent State — Shared Memory (`tool_context.state`)

> [!CAUTION]
> **State is the "glue" that binds agents together.** Without state, downstream agents wouldn't know what the user asked or what researches were previously done.

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

---

### 🔷 Tools Explained

#### ✅ `add_prompt_to_state` — Custom Python Function Tool
- **Purpose:** Saves the user's message into `state["PROMPT"]`.
- **Why needed?** The `root_agent` handles the initial greeting. This tool ensures that the actual question doesn't get "lost" when moving to the researcher.

#### ✅ `LangchainTool` — Wikipedia Tool
- **Purpose:** An **adapter** that lets you use existing LangChain ecosystem tools inside the ADK framework.

---

## 4. Deployment Concepts

### 🔷 `uvx` — Run CLI Tools Without Installing
`uvx` runs a command **ephemerally**.
> [!TIP]
> Use `uvx` whenever you want to test or run a tool once without cluttering your system with global installs.

---

### 🔷 `adk deploy cloud_run` — Automation
This single command handles:
1.  **Cloud Build** images your code.
2.  **Artifact Registry** stores it.
3.  **Cloud Run** spins up the service.

---

### 🔷 Service Account & Security
> [!IMPORTANT]
> Always use a **dedicated service account** for cloud services. This limits potential damage if a service is compromised (Least Privilege).

---

## 6. Clean Up — Why It Matters

> [!WARNING]
> **Costs accrue while services are live.**
> 1. Delete the **Cloud Run service**.
> 2. Delete the **Artifact Registry repository** (images take up storage $).
