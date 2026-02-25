# Same M4. Same 64GB. Different Front Door.

> *Same origin story as the Open WebUI build — M4 Mini Pro, 64GB of unified RAM, the need for a single URL that handles coding, compliance docs, Splunk queries, image generation, music, voice cloning, and everything in between — but this time, the interface is LibreChat.*

Two guides. One machine. Pick your UI.

---

## What You're Building

One URL. Everything on-device. Zero subscriptions. Zero API keys. Zero data leaving your machine.

Type `http://localhost:8080` and you get LibreChat — a polished, conversation-centric interface that:

- **Routes your code questions** to the right model automatically — PowerShell, Python, BigFix fixlets, all handled without you picking a model
- **Loads specialized models for Splunk** when you paste SPL — validates queries against a live instance if you have one
- **Answers questions about your policy documents** via RAG — upload a PDF, ask a question, get an answer sourced from your own docs
- **Generates images** from text with FLUX.1-schnell running on your M4's Neural Engine, link appears inline
- **Composes music** from a prompt — specify the genre, mood, and BPM, AudioCraft does the rest locally
- **Clones and synthesizes voice** in 14 languages using XTTS-v2, no ElevenLabs account needed
- **Fetches live web content** directly into chat — API docs, vendor portals, Cloudflare-protected pages via Scrapling
- **Remembers things across sessions** with a persistent knowledge graph — your infrastructure context survives reboots
- **Builds structured documents and slide decks** from chat prompts, complete with audit metadata

The model selection happens the same way it does in the Open WebUI build — a FastAPI router intercepts every request, scores your input against keyword rules, and routes to the right model before Ollama ever sees it. The difference is LibreChat's native MCP stdio support means no `mcpo` proxy layer for the core tool servers. They spawn directly.

**This is a 15-phase step-by-step guide** from a freshly formatted Mac to a running stack. Every phase has complete configuration, working code, and a troubleshooting section. Nothing is skipped.

---

## By the Numbers

| | |
|---|---|
| **15** phases — fresh Mac to full running stack |
| **15+** local LLM models with the same router and custom Modelfiles |
| **5** generation services — images, music, voice, documents, slides |
| **7** native MCP tool servers — no proxy, LibreChat spawns them directly |
| **~55GB** RAM at full — comfortably inside your 64GB |
| **$0/month** in API costs after setup |

---

## Open WebUI vs. LibreChat — Why This Guide Exists

These two builds run on the same hardware with the same models and the same router. The difference is the interface and how tools are wired in.

| | Open WebUI | LibreChat |
|---|---|---|
| **Configuration** | GUI-based admin panel | `librechat.yaml` — version-controlled, repeatable |
| **MCP tool servers** | Needs `mcpo` HTTP proxy to expose stdio MCPs | Native stdio spawn — servers run inside the API container |
| **Custom agents** | Open WebUI Tools system | Agent Builder with OpenAPI backends |
| **Model display names** | Direct Ollama model names | `modelSpecs` — clean display names over the same routing layer |
| **Conversation style** | Single-window multi-modal | Thread-based, shareable, bookmark-friendly |

Neither is better. They're different tools for different preferences. This repo is for the LibreChat path.

---

## What You Need

| Component | Requirement |
|-----------|-------------|
| Hardware | Apple M4 Pro Mac Mini |
| Memory | 64GB unified memory |
| OS | macOS Sequoia |
| Package manager | Homebrew |
| Python | 3.11 (global virtual environment) |
| Container runtime | Docker Desktop |

> **On memory:** Full stack runs at ~55GB. The generation services (ComfyUI, MusicGen, Voice Clone) load on demand, not at startup. The base stack without generation services is substantially lighter.

---

## Start Here

| Document | What It Covers |
|----------|----------------|
| [**LibreChat Stack Setup Guide v1.2**](docs/LibreChat_Stack_Setup_Guide_v1_2.md) | All 15 phases from fresh format to running stack — complete commands, configs, and troubleshooting |
| [**MCP Core Servers v1.1**](docs/LC_MCP_Core_Servers_v1_1.md) | The native stdio MCP tool layer — filesystem, memory, SQLite, git, sequential thinking, time, and Microsoft Learn |
| [**MCP Scrapling v1.1**](docs/LC_MCP_Scrapling_v1_1.md) | Live web fetching via streamable-http MCP — three escalation tiers from fast HTTP to stealth anti-bot bypass |
| [**Production Readiness Review v1.2**](docs/LC_Production_Readiness_Review_v1_2.md) | Full lifecycle verification — all outstanding findings resolved, production-ready sign-off |

---

## How It's Laid Out

```
                        http://localhost:8080
                                 │
                        ┌────────▼────────┐
                        │      Caddy      │  One URL to rule them all
                        │     :8080       │  Routes everything · Serves all generated files
                        └────────┬────────┘
               ┌─────────────────┼─────────────────┐
               │                 │                 │
       ┌───────▼───────┐ ┌───────▼───────┐ ┌───────▼───────┐
       │   LibreChat   │ │    Router     │ │   SearxNG     │
       │    :3080      │ │    :8000      │ │    :8888      │
       │  Chat + Agents│ │  Model Brain  │ │  Local Search │
       └───────┬───────┘ └───────┬───────┘ └───────────────┘
               │                 │
       ┌───────▼───────┐ ┌───────▼─────────────────────────┐
       │   ChromaDB    │ │   Ollama  :11434                 │
       │    :8500      │ │   15+ models · M4 MPS            │
       │  Your Docs    │ │   Custom Modelfiles              │
       └───────────────┘ └──────────────────────────────────┘

  Generation — Agent Builder calls these, link appears in chat
    ComfyUI         FLUX.1-schnell (~24GB)    ──►  ~/AI_Output/images/
    MusicGen        AudioCraft Large          ──►  ~/AI_Output/audio/
    Voice Clone     XTTS-v2 (~2GB)            ──►  ~/AI_Output/voice/
    DocGen :8002    Pandoc + Typst            ──►  ~/AI_Output/docs/
    Presenton :5000 PPTX/PDF slides           ──►  ~/AI_Output/slides/

  MCP Tool Layer — native stdio spawn, no proxy required (~205MB idle)
    Scrapling  :8900   Web fetch · HTTP → Chromium → Stealth bypass
    Filesystem         Read/write local files
    Memory             Persistent knowledge graph
    SQLite             Query local databases from chat
    Git                Repo inspection, diffs, commit history
    Sequential Thinking  Structured step-by-step reasoning
    Time               Timezone math and date arithmetic
    MS Learn   remote  Official Microsoft documentation
```

---

## The 15 Phases

| Phase | What Happens |
|-------|-------------|
| 1 | System foundation — Homebrew, Python 3.11, PyTorch with M4 MPS, core CLI tools |
| 2 | Ollama + 15+ models — `bigfix-expert` and `splunk-secops` custom Modelfiles included |
| 3 | Caddy — single entry point at `:8080`, routes all traffic, serves generated output files |
| 4 | Docker stack — LibreChat API + UI, ChromaDB vector store, SearxNG local search |
| 5 | `librechat.yaml` — complete configuration: endpoints, model specs, MCP servers, agent tools |
| 6 | Model Router — the FastAPI brain at `:8000`, routes by keyword scoring before Ollama sees it |
| 7 | ComfyUI — FLUX.1-schnell image generation wired as an Agent Builder OpenAPI backend |
| 8 | MusicGen — AudioCraft with lazy-load, exposed via Agent Builder |
| 9 | Voice Clone — XTTS-v2 with speaker embedding cache, wired via Agent Builder |
| 10 | Document Generation — DocGen API (Pandoc + Typst) + Presenton slides as Agent Builder backends |
| 11 | Policy & Procedure RAG — document upload via API, queried through LibreChat conversations |
| 12 | MCP Core Servers — filesystem, memory, SQLite, git, sequential thinking, time via native stdio |
| 13 | Scrapling MCP — streamable-http web fetching wired into `librechat.yaml` |
| 14 | Master Launcher — boot script with startup order, model pre-warming, macOS LaunchAgent |
| 15 | Smoke Test Suite — end-to-end verification of every service and routing path |

---

## The Model Router — Same Brain, Different UI

The routing logic is identical to the Open WebUI build. A FastAPI proxy sits between LibreChat and Ollama, scoring every request against keyword rules before forwarding:

| You type... | Router loads... | How |
|-----------|----------------|-----|
| Anything BigFix, fixlets, relevance expressions | `bigfix-expert` | Regex keyword hit |
| Splunk SPL, tstats, CIM, ES notables | `splunk-secops` | Regex keyword hit |
| Write Python/PowerShell, debug this function | `qwen3-coder:30b` | Coding intent detected |
| Analyze, research, think through | `deepseek-r1:32b` | Reasoning intent detected |
| `@model:llama3.3:70b` anywhere in your message | `llama3.3:70b` | Manual override, always wins |

LibreChat's `modelSpecs` configuration wraps these routing targets with clean display names in the model selector dropdown. The underlying routing is the same — `modelSpecs` is presentation layer only.

---

## The MCP Layer — Native in LibreChat

LibreChat spawns stdio MCP servers directly from the `librechat.yaml` config — no HTTP proxy wrapper required. The API container manages the process lifecycle.

| Server | What It Does |
|--------|-------------|
| **Filesystem** | Reads and writes local files mounted into the container |
| **Memory** | Persistent knowledge graph in JSON — survives restarts |
| **SQLite** | Full SQL queries against local databases from chat |
| **Git** | Inspect repos, read diffs, review commit history |
| **Sequential Thinking** | Structured step-by-step reasoning with revision capability |
| **Time** | Timezone conversions and date arithmetic |
| **Scrapling** | Live web fetching — HTTP, Chromium, or stealth Firefox depending on the target |
| **Microsoft Learn** | Remote endpoint — official MS docs, zero local install |

Total idle memory: ~205MB. Native stdio spawn, no proxy overhead.

---

## Agent Builder — Generation Services as OpenAPI Backends

LibreChat exposes generation capabilities through its Agent Builder rather than the Tools system. Each service runs as an independent FastAPI backend and is registered as an Agent with an OpenAPI spec:

- **Image generation**: Prompt to FLUX.1-schnell, URL returned inline
- **Music generation**: Text prompt → AudioCraft → audio file link in chat
- **Voice cloning**: Upload a sample, type what to say, get a file back
- **Document generation**: Describe the doc, get a DOCX/PDF download link
- **Slide generation**: Describe your deck, get a PPTX from Presenton

Each Agent has specific instructions, tool access, and model assignment configured in the Agent Builder UI. The [Setup Guide](docs/LibreChat_Stack_Setup_Guide_v1_2.md) covers all of it.

---

## Everything Stays on Your Machine

No subscriptions. No API keys for AI services. No telemetry to vendors. No model inference leaving the box. Your policy documents, your code, your voice samples, your findings database — none of it goes anywhere.

The `librechat.yaml` config is version-controlled. The entire stack is reproducible from a clean machine using this guide. There are no manual clicks required to reconstruct it — it's all in files.

---

## Production Readiness

The [Production Readiness Review](docs/LC_Production_Readiness_Review_v1_2.md) documents a full lifecycle verification pass against the v1.2 guide. All high and medium severity findings from the v1.1 review are resolved. The document suite is cleared for single-user M4 Mac Mini deployment.

---

## Versioning

| Document | Version | Update |
|----------|---------|--------|
| LibreChat Stack Setup Guide | **v1.2** | Feb 2026 — all v1.1 findings resolved |
| MCP Core Servers | **v1.1** | Feb 2026 |
| MCP Scrapling | **v1.1** | Feb 2026 — SSRF awareness added |
| Production Readiness Review | **v1.2** | Feb 2026 — PASS verdict |

---

## License

MIT — see [LICENSE](LICENSE). Build on it, adapt it, share it.
