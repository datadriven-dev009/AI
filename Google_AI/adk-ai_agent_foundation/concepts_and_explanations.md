# ADK AI Agent Foundation — Concepts & Explanations

---

## What is an AI Agent?

> A smart program that **perceives, decides, and acts** to achieve goals — without direct human control.

- Uses an **LLM as its "brain"** (reasoning & understanding)
- Can process **text, images, sounds**
- Creates a **plan** and executes tasks autonomously
- Think of it as a **proactive personal assistant**

---

## What is ADK?

**Agent Development Kit (ADK)** is Google's framework for building AI agents — even without deep AI expertise.

---

## Why `uv` Instead of `venv`?

> `uv` is a **Rust-based** Python package manager — significantly faster than standard `pip` + `venv`.

- Handles virtual environments + package installs in one tool
- Drop-in replacement — commands are nearly identical

---

## The 3 Core Files

### `agent.py` — The Brain
| Parameter | Purpose |
|-----------|---------|
| `name` | Unique ID for the agent |
| **`model`** | The LLM used as the brain (e.g., `gemini-2.5-flash`) |
| `description` | Human-readable summary (also used in multi-agent systems) |
| **`instruction`** | System prompt — shapes agent's **persona & behavior** |

### `__init__.py` — The Package Marker
- Makes the folder a **Python package**
- Lets ADK discover and load your agent
- Even if nearly empty, **its presence is required**

### `.env` — The Config Store
| Variable | Purpose |
|----------|---------|
| `GOOGLE_GENAI_USE_VERTEXAI` | Use Vertex AI as backend |
| `GOOGLE_CLOUD_PROJECT` | Your GCP project ID |
| `GOOGLE_CLOUD_LOCATION` | Region for Vertex AI (e.g., `us-central1`) |

> ⚠️ Never commit `.env` to version control — it holds sensitive credentials.

---

## Terminal vs Web UI

| Feature | `adk run` (Terminal) | `adk web` (Web UI) |
|---------|---------------------|-------------------|
| Interface | Plain text | Chat app UI |
| Markdown | Raw (hard to read) | **Rendered correctly** |
| Debugging | ❌ | ✅ View message events & agent state |
| Best for | Quick testing | Development & debugging |

---

## Key Concepts Summary

```
LLM (Gemini)  →  Brain / Reasoning engine
ADK           →  Framework to build & run agents
agent.py      →  Agent definition
instruction   →  Controls agent behavior (system prompt)
adk web       →  Local dev UI at localhost:8000
Vertex AI     →  Google Cloud managed AI backend
```

---

## Next Step

> 🚀 **Building AI Agents with ADK: Empowering with Tools**  
> Add real-time info access and external service integrations to your agent.
