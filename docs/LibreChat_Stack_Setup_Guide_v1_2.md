# LibreChat AI Stack Setup Guide v1.2
### Complete M4 Mac Mini Pro (64GB) — LibreChat Edition
**Target Machine:** Apple M4 Pro · 64GB Unified Memory · macOS Sequoia  
**Primary Interface:** `http://localhost:8080` — everything through one URL  
**UI:** LibreChat (replaces Open WebUI)  
**Companion to:** M4 AI Stack Setup Guide v6.2  
**Document version:** 1.2 (Feb 24, 2026)

---

## What This Guide Is

This is a **complete, standalone setup guide** for running the M4 AI Stack with LibreChat as the UI instead of Open WebUI. It is not a diff — it is the full path from a fresh format to a working stack, covering every phase that changes and every phase that stays the same.

**If you are migrating from an existing Open WebUI installation**, see the Migration Checklist at the end of this document.

---

## Why LibreChat vs Open WebUI

| Dimension | Open WebUI | LibreChat |
|---|---|---|
| **UI familiarity** | Original interface | ChatGPT-identical layout |
| **MCP servers (stdio)** | Requires mcpo proxy | Native — spawns directly |
| **MCP servers (HTTP)** | Native Streamable HTTP | Native Streamable HTTP |
| **Agent builder** | Limited | Full no-code agent builder |
| **Web search** | Plugin-based | Native YAML config |
| **RAG** | Built-in ChromaDB | Separate RAG API + pgvector |
| **Multi-user auth** | Basic | OAuth/OIDC/LDAP/SAML |
| **Configuration** | Admin GUI | YAML files (auditable, versionable) |
| **Conversation forking** | No | Yes |
| **Prompt library** | Limited | Full template library with variables |
| **Infrastructure deps** | SQLite (built-in) | MongoDB + Redis + Meilisearch |
| **Memory (UI container)** | ~1.5GB | ~800MB |
| **mcpo proxy needed** | Yes | **No — eliminated** |

**Net memory change from Open WebUI stack:** approximately -300MB despite adding MongoDB and Meilisearch, because LibreChat eliminates mcpo and its 6 child processes, and the UI container itself is lighter.

---

## Architecture

```
http://localhost:8080  (Caddy — single entry point)
        │
        ├── /           → LibreChat :3080
        ├── /audio/*    → ~/AI_Output/audio/
        ├── /voice/*    → ~/AI_Output/voice/
        ├── /images/*   → ~/AI_Output/images/
        ├── /docs/*     → ~/AI_Output/docs/
        ├── /slides/*   → ~/AI_Output/slides/
        └── /router/*   → Model Router :8000
                │
                │  workspace virtual models + regex + @model: override
                │  VRAM residency manager  |  OOM fallback chain
                │
        Ollama :11434

LibreChat ← MongoDB :27017 (conversations, users, agents)
          ← Redis :6379 (session cache)
          ← Meilisearch :7700 (full-text conversation search)
          ← RAG API :8090 (document indexing, optional)
          │
          ├── MCP: Filesystem (stdio, spawned by LibreChat container)
          ├── MCP: Memory (stdio, spawned by LibreChat container)
          ├── MCP: SQLite (stdio, spawned by LibreChat container)
          ├── MCP: Git (stdio, spawned by LibreChat container)
          ├── MCP: Sequential Thinking (stdio, spawned by LibreChat container)
          ├── MCP: Time (stdio, spawned by LibreChat container)
          ├── MCP: Scrapling :8900 (streamable-http, external process)
          └── MCP: Microsoft Learn (streamable-http, remote)

SearxNG :8888 (web search backend — unchanged from M4 stack)
ComfyUI :8188 (image generation — unchanged)
Music API :8001, Voice API :5002, DocGen API :8002 (on-demand — unchanged)
Presenton :5000 (presentation generation — unchanged)
```

**Key differences from Open WebUI architecture:**
- LibreChat spawns stdio MCP servers as child processes inside its container — the entire `:9000` mcpo proxy is eliminated
- MongoDB + Redis + Meilisearch replace Open WebUI's built-in SQLite and basic search
- Scrapling (`:8900`) remains an external native HTTP MCP server — same as before
- All backend services (Ollama, ComfyUI, Phase 5 router, SearxNG, etc.) are **completely unchanged**

---

## Phase Map — What Changes, What Doesn't

| Phase | Open WebUI Path | LibreChat Path | Status |
|---|---|---|---|
| Phase 1 — System Foundation | ← | Same | ✅ No change |
| Phase 2 — Ollama | ← | Same | ✅ No change |
| Phase 3 — Caddy | Upstream `:3000` | Upstream `:3080` | ⚠️ 1-line change |
| Phase 4 — Docker Stack | Open WebUI + ChromaDB + SearxNG | **LibreChat + MongoDB + Redis + Meilisearch + SearxNG** | 🔄 Full replacement |
| Phase 5 — Model Router | ← | Same | ✅ No change |
| Phase 6 — ComfyUI | Native OWUI integration | **ComfyUI Action/Agent + proxy** | ⚠️ Action required |
| Phase 7 — MusicGen | ← | Same | ✅ No change |
| Phase 8 — Voice Clone | ← | Same | ✅ No change |
| Phase 9 — Open WebUI Tools | Python tools in OWUI admin | **Agents + Actions (OpenAPI) in LibreChat** | 🔄 Different approach |
| Phase 10 — Policy RAG | OWUI upload + ChromaDB | **RAG API + pgvector or ChromaDB** | 🔄 Different backend |
| Phase 11 — DocGen API | ← | Same (API unchanged) | ✅ No change |
| Phase 12 — Launcher | Starts OWUI + mcpo | **Starts LibreChat, removes mcpo block** | ⚠️ Minor update |
| Phase 13 — Smoke Test | OWUI endpoints | **LibreChat endpoints** | ⚠️ URL updates |
| Phase 14 — Personal Mode | ← | Same (router-side) | ✅ No change |
| Phase 15 — MCP Servers | mcpo proxy (`:9000`) + Scrapling | **Native stdio in librechat.yaml + Scrapling** | 🔄 Architecture change |

---

## PHASE 1 — System Foundation

**No changes.** Follow Phase 1 of the M4 AI Stack Setup Guide v6.2 exactly.

---

## PHASE 2 — Ollama (LLM Runtime)

**No changes.** Follow Phase 2 of the M4 AI Stack Setup Guide v6.2 exactly. All model pulls, custom Modelfiles, and configuration are identical.

---

## PHASE 3 — Caddy (One-Line Change)

Follow Phase 3 of the M4 AI Stack Setup Guide v6.2 with one change: the Open WebUI upstream on `:3000` becomes LibreChat on `:3080`.

In `~/ai-stack/caddy/Caddyfile`, change:

```caddyfile
# Find this line in the handle block at the bottom:
reverse_proxy localhost:3000

# Change it to:
reverse_proxy localhost:3080
```

Everything else in the Caddyfile is identical — all file server blocks (`/audio/*`, `/voice/*`, `/images/*`, `/docs/*`, `/slides/*`) and the `/router/*` proxy remain unchanged.

If Caddy is already running:
```bash
caddy reload --config ~/ai-stack/caddy/Caddyfile --adapter caddyfile
echo "Caddy now points to LibreChat :3080"
```

---

## PHASE 4 — Docker Stack (Full Replacement)

This phase replaces Open WebUI + ChromaDB with LibreChat + MongoDB + Redis + Meilisearch. SearxNG is retained as-is.

### Step 4.1 — Create Directories

```bash
# These are needed by both SearxNG (carried over) and LibreChat's MCP servers
mkdir -p ~/AI_Output/{audio,voice,images,docs,slides}
mkdir -p ~/ai-stack/searxng
mkdir -p ~/voice_samples

# MCP filesystem targets — created here so they exist before LibreChat starts
mkdir -p ~/Projects ~/Scripts ~/data ~/.mcp-memory

# LibreChat persistent MCP data
mkdir -p ~/LibreChat/data/mcp-memory
mkdir -p ~/LibreChat/data/mcp-sqlite

# Initialize SQLite with a valid database header (prevents "file is not a database"
# error on first read query — touch creates a 0-byte file, not a valid SQLite db)
sqlite3 ~/LibreChat/data/mcp-sqlite/ops.db "SELECT 1;" 2>/dev/null \
  && echo "✓ SQLite database initialized" \
  || { touch ~/LibreChat/data/mcp-sqlite/ops.db; echo "⚠ sqlite3 not found — created placeholder (writes will initialize the db)"; }

# Verify ops.db placeholder has correct permissions
ls -la ~/LibreChat/data/mcp-sqlite/ops.db
```

> **SQLite initialization:** The `sqlite3` command above creates a properly structured database file (~8KB with a valid SQLite header). This prevents `"file is not a database"` errors when the first MCP interaction is a read (e.g., `list_tables`). `sqlite3` is pre-installed on macOS Sequoia.

### Step 4.2 — SearxNG Configuration

SearxNG is carried over from the M4 stack without modification. The settings file is identical.

**If migrating from an existing M4 stack:** Your `~/ai-stack/searxng/settings.yml` is already correct. SearxNG will continue running from `~/ai-stack/docker-compose.yml` alongside any remaining M4 services, or you can migrate it to the standalone file below.

**For fresh LibreChat-only installs (no prior M4 stack):** Create a standalone SearxNG compose file. The LibreChat launcher uses this by default.

```bash
mkdir -p ~/ai-stack/searxng

# Standalone SearxNG compose for LibreChat path
# (M4 stack migrants: this file is optional — your existing compose works too)
cat > ~/ai-stack/docker-compose-searxng.yml << 'EOF'
services:
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    ports:
      - "127.0.0.1:8888:8080"
    volumes:
      - ./searxng:/etc/searxng
    environment:
      - SEARXNG_BASE_URL=http://searxng:8080
    restart: unless-stopped
EOF
```

Create `~/ai-stack/searxng/settings.yml` — copy from Phase 4.2 of the M4 AI Stack Setup Guide v6.2 verbatim, with one required addition:

```yaml
search:
  formats:
    - html
    - json   # ← Required for LibreChat's native SearxNG web search integration
```

### Step 4.3 — Clone LibreChat

```bash
cd ~
git clone https://github.com/danny-avila/LibreChat.git
cd LibreChat

# Pin to latest stable release
git fetch --tags
git checkout $(git describe --tags $(git rev-list --tags --max-count=1))

# Create .env from template
cp .env.example .env
```

### Step 4.4 — Configure `.env`

The `.env` file is generated in a single command using an **unquoted heredoc** so that `$(openssl rand -hex 32)` expands at write time. Never copy a template with placeholder text — the keys must be real random values.

```bash
# IMPORTANT: This heredoc is UNQUOTED (<< ENVEOF, not << 'ENVEOF').
# Bash expands $(openssl rand ...) at write time, producing real keys.
# Verify after running: grep CREDS_KEY ~/LibreChat/.env
# It should show a 64-character hex string, not the literal $(openssl ...) text.

cat > ~/LibreChat/.env << ENVEOF
# ── Core ────────────────────────────────────────────
HOST=0.0.0.0
PORT=3080
MONGO_URI=mongodb://mongodb:27017/LibreChat

# ── Endpoints ────────────────────────────────────────
ENDPOINTS=custom,agents

# ── Search ───────────────────────────────────────────
SEARCH=true
MEILI_HOST=http://meilisearch:7700
MEILI_MASTER_KEY=$(openssl rand -hex 32)

# ── SearxNG ──────────────────────────────────────────
SEARXNG_INSTANCE_URL=http://host.docker.internal:8888

# ── Redis ────────────────────────────────────────────
REDIS_URI=redis://redis:6379

# ── RAG API (optional) ───────────────────────────────
RAG_PORT=8090
RAG_API_URL=http://rag_api:8090

# ── Credentials (auto-generated) ─────────────────────
CREDS_KEY=$(openssl rand -hex 32)
CREDS_IV=$(openssl rand -hex 16)
JWT_SECRET=$(openssl rand -hex 32)
JWT_REFRESH_SECRET=$(openssl rand -hex 32)

# ── Session ──────────────────────────────────────────
SESSION_EXPIRY=900000
REFRESH_TOKEN_EXPIRY=604800000

# ── Registration ─────────────────────────────────────
ALLOW_REGISTRATION=true
ALLOW_SOCIAL_LOGIN=false
ALLOW_SOCIAL_REGISTRATION=false

# ── Node memory + host path for Docker Compose ───────
NODE_OPTIONS=--max-old-space-size=4096
HOST_HOME=$HOME
ENVEOF

echo "✓ .env written — verifying keys were generated:"
grep -E "CREDS_KEY|JWT_SECRET|MEILI_MASTER_KEY" ~/LibreChat/.env | cut -c1-40
# Each line should show a real hex string, not literal $(openssl ...) text
```

### Step 4.5 — Create `librechat.yaml`

This is the heart of the LibreChat configuration. It replaces Open WebUI's admin panel settings, workspace model assignments, MCP server connections, and web search configuration.

```bash
cat > ~/LibreChat/librechat.yaml << 'LCYAML'
version: 1.3.4
cache: true

# ── Endpoints: Ollama via Phase 5 Router ──────────────────────────────
# All traffic goes through the Phase 5 router at :8000, not directly to Ollama.
# The router handles workspace routing, VRAM management, and OOM fallback.
# Virtual model names here MUST match the workspace names in the router.
endpoints:
  custom:
    - name: "M4 Stack"
      apiKey: "ollama"
      # Phase 5 router — all workspace virtual models route through here
      baseURL: "http://host.docker.internal:8000/v1/"
      models:
        default:
          - "auto"
          - "auto-coding"
          - "auto-agentic-code"
          - "auto-reasoning"
          - "auto-reasoning-oss"
          - "auto-glm"
          - "auto-fast"
          - "auto-rag"
          - "auto-bigfix"
          - "auto-splunk"
          - "auto-docgen"
          - "auto-slides"
        # fetch: false — DO NOT change this to true.
        # The Phase 5 router's /v1/models endpoint returns virtual model names
        # (auto, auto-bigfix, auto-splunk, etc.), not raw Ollama model names.
        # Setting fetch: true would override this explicit list with qwen3:32b,
        # deepseek-r1:32b, etc., breaking workspace routing and Model Spec names.
        fetch: false
      titleConvo: true
      titleModel: "current_model"
      summarize: false
      summaryModel: "current_model"
      modelDisplayLabel: "M4 Stack"

    # Direct Ollama access — bypass router for raw model selection
    - name: "Ollama Direct"
      apiKey: "ollama"
      baseURL: "http://host.docker.internal:11434/v1/"
      models:
        default: ["qwen3:32b"]
        fetch: true
      titleConvo: true
      titleModel: "current_model"
      modelDisplayLabel: "Ollama Direct"

# ── Web Search: SearxNG ──────────────────────────────────────────────
# Points to existing SearxNG container — no changes to SearxNG itself.
webSearch:
  searchProvider: "searxng"
  searxngInstanceUrl: "${SEARXNG_INSTANCE_URL}"
  safeSearch: 0

# ── MCP Servers ──────────────────────────────────────────────────────
# LibreChat spawns stdio servers directly as child processes of the API container.
# This completely replaces the mcpo proxy from the Open WebUI stack.
# No :9000 port, no mcpo process, no conversion layer.
mcpServers:
  filesystem:
    title: "Filesystem"
    description: "Read/write/search local project files, scripts, compliance docs, AI outputs"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-filesystem"
      - "/host-home/Projects"
      - "/host-home/Scripts"
      - "/host-home/Documents"
      - "/host-home/AI_Output"
    # Paths are container-side mount points configured in docker-compose.override.yml

  memory:
    title: "Memory"
    description: "Persistent knowledge graph — entities, relationships, observations across sessions"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-memory"
    env:
      MEMORY_FILE_PATH: "/app/data/mcp-memory/memory.json"

  sqlite:
    title: "SQLite"
    description: "Query local databases — findings tracker, asset inventory, analytics"
    command: "uvx"
    args:
      - "mcp-server-sqlite"
      - "--db-path"
      - "/app/data/mcp-sqlite/ops.db"

  git:
    title: "Git"
    description: "Inspect repos — branches, diffs, commit history, code review"
    command: "uvx"
    args:
      - "mcp-server-git"
      - "--repository"
      - "/host-home/Projects"

  sequential-thinking:
    title: "Sequential Thinking"
    description: "Structured step-by-step reasoning with revision and branching"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-sequential-thinking"

  time:
    title: "Time"
    description: "Timezone conversions and date math across TX/NY/CA regions"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-time"
      - "--local-timezone=America/Chicago"

  scrapling:
    title: "Scrapling Web Fetcher"
    description: "Fetch and parse web content with anti-bot bypass and CSS selector targeting"
    type: streamable-http
    url: "http://host.docker.internal:8900/mcp"

  microsoft-learn:
    title: "Microsoft Learn"
    description: "Official Microsoft documentation grounding — zero local install"
    type: streamable-http
    url: "https://learn.microsoft.com/api/mcp"

# ── Agent Configuration ──────────────────────────────────────────────
agents:
  webSearch: true
  fileSearch: true
  recursionLimit: 25
  maxRecursionLimit: 50

# ── Model Specs (workspace equivalents for LibreChat's model dropdown) ──
# These create named presets in the UI — equivalent to Open WebUI workspaces.
modelSpecs:
  enforce: false    # false = user can still select other models manually
  prioritize: true  # show specs at top of model list
  list:
    - name: "auto"
      label: "General Assistant"
      description: "Default workspace — routes to qwen3:32b via regex scoring"
      preset:
        endpoint: "M4 Stack"
        model: "auto"

    - name: "auto-coding"
      label: "Code Assistant"
      description: "Routes to qwen3-coder:30b for programming tasks"
      preset:
        endpoint: "M4 Stack"
        model: "auto-coding"

    - name: "auto-agentic-code"
      label: "Agentic Coder"
      description: "Routes to devstral-small-2 for multi-file editing and codebase exploration"
      preset:
        endpoint: "M4 Stack"
        model: "auto-agentic-code"

    - name: "auto-reasoning"
      label: "Deep Reasoning"
      description: "Routes to deepseek-r1:32b for complex analysis"
      preset:
        endpoint: "M4 Stack"
        model: "auto-reasoning"

    - name: "auto-reasoning-oss"
      label: "OpenAI Reasoning"
      description: "Routes to gpt-oss:20b — o3-mini class reasoning, tool use"
      preset:
        endpoint: "M4 Stack"
        model: "auto-reasoning-oss"

    - name: "auto-bigfix"
      label: "BigFix Expert"
      description: "Routes to bigfix-expert for BigFix relevance and action scripts"
      preset:
        endpoint: "M4 Stack"
        model: "auto-bigfix"

    - name: "auto-splunk"
      label: "Splunk SecOps"
      description: "Routes to splunk-secops for SPL and Splunk Enterprise Security"
      preset:
        endpoint: "M4 Stack"
        model: "auto-splunk"

    - name: "auto-fast"
      label: "Quick Queries"
      description: "Always qwen3:8b — instant responses"
      preset:
        endpoint: "M4 Stack"
        model: "auto-fast"
LCYAML
```

> **How Model Specs relate to router routing (important for Open WebUI migrants):**
>
> LibreChat Model Specs are **UI presets** — they populate the model dropdown with friendly names. They are not workspace locks. With `enforce: false`, users can still manually type any model name.
>
> The actual routing enforcement lives entirely in the **Phase 5 router**. When you select `auto-bigfix` from a Model Spec, LibreChat sends `model: "auto-bigfix"` to the router, which matches it against its workspace routing table and routes deterministically to `bigfix-expert` — with VRAM management and OOM fallbacks. This is functionally equivalent to Open WebUI's workspace lock behavior.
>
> `enforce: false` means users *can* type model names manually. Any virtual model name the router recognizes (`auto`, `auto-bigfix`, `auto-splunk`, etc.) routes correctly. Unknown names pass directly to Ollama, bypassing workspace routing. For a single-user setup this is fine. Set `enforce: true` for team deployments where you want the dropdown to be the only selection mechanism.
>
> **Bottom line:** The router does the enforcing. LibreChat's Model Spec is the label on the door. If `auto-bigfix` appears in the dropdown and the router has it in its routing table, routing is correctly locked — the UI `enforce` setting only controls whether users can manually type other names, not whether the router routes correctly when the spec is selected.

### Step 4.6 — Create `docker-compose.override.yml`

This file extends LibreChat's base Docker Compose. It uses `${HOST_HOME}` which Docker Compose reads from `~/LibreChat/.env` (populated in Step 4.4) — not from the shell environment. This ensures correct path resolution whether the stack is started manually or via the LaunchAgent.

The npm cache volume (`npm_cache`) ensures that `npx`-spawned MCP servers do not hit the npm registry on every LibreChat restart, eliminating 2-5 second cold-start latency on tool calls.

```bash
cat > ~/LibreChat/docker-compose.override.yml << 'LCOVERRIDE'
services:
  api:
    volumes:
      # librechat.yaml configuration
      - type: bind
        source: ./librechat.yaml
        target: /app/librechat.yaml

      # Home directory mounts for MCP Filesystem server
      # ${HOST_HOME} is read from ~/LibreChat/.env — reliable in all launch contexts
      - type: bind
        source: ${HOST_HOME}/Projects
        target: /host-home/Projects
      - type: bind
        source: ${HOST_HOME}/Scripts
        target: /host-home/Scripts
      - type: bind
        source: ${HOST_HOME}/Documents
        target: /host-home/Documents
      - type: bind
        source: ${HOST_HOME}/AI_Output
        target: /host-home/AI_Output

      # Persistent MCP data directories
      - ./data/mcp-memory:/app/data/mcp-memory
      - ./data/mcp-sqlite:/app/data/mcp-sqlite

      # npm cache — prevents npx from hitting the registry on every container restart.
      # Without this, each MCP server spawn (npx -y @modelcontextprotocol/...) adds
      # 2-5 seconds latency while npx checks for the latest version.
      - npm_cache:/root/.npm

    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - NODE_OPTIONS=--max-old-space-size=4096

volumes:
  npm_cache:
    driver: local
LCOVERRIDE
```

> **Why `${HOST_HOME}` instead of `${HOME}`?** Docker Compose resolves variables from its own `.env` file, not from the calling shell's environment. `${HOME}` works when you run `docker compose up` interactively (shell env is passed through), but may resolve incorrectly when the LaunchAgent runs the launcher at login. `HOST_HOME` is explicitly set to `$HOME` during Step 4.4 `.env` generation, making it reliable in all launch contexts.

### Step 4.7 — Launch the Stack

```bash
cd ~/LibreChat
docker compose up -d

# Watch logs until ready (~60s on first start — MongoDB and Meilisearch need to initialize)
docker compose logs -f api --tail=30
# Look for: "Server listening on port 3080"
# Ctrl+C once you see it
```

### Step 4.8 — Verify and Create Account

```bash
# LibreChat API
curl -s -o /dev/null -w "%{http_code}" http://localhost:3080
# Expected: 200

# Via Caddy (your primary entry point)
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080
# Expected: 200

# MongoDB
docker compose exec mongodb mongosh --eval "db.runCommand({ping:1})" --quiet
# Expected: { ok: 1 }

# Meilisearch
curl -s http://localhost:7700/health
# Expected: {"status":"available"}

# Redis
docker compose exec redis redis-cli ping
# Expected: PONG
```

Navigate to `http://localhost:8080` → **Register** → create your account. Credentials are stored locally in MongoDB.

### ✅ Phase 4 Preflight Check

```bash
echo "--- LibreChat Stack Health ---"
curl -s -o /dev/null -w "LibreChat (3080): %{http_code}\n" http://localhost:3080
curl -s -o /dev/null -w "Via Caddy (8080): %{http_code}\n" http://localhost:8080
curl -s http://localhost:7700/health | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Meilisearch: {d[\"status\"]}')"
docker compose -f ~/LibreChat/docker-compose.yml exec redis redis-cli ping 2>/dev/null && echo "Redis: PONG"

echo "--- SearxNG ---"
curl -s "http://localhost:8888/search?q=NERC+CIP&format=json" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); n=len(d.get('results',[])); print(f'SearxNG: {n} results'); exit(0 if n>0 else 1)"

echo "--- MCP Servers ---"
echo "(MCP servers are spawned inside the LibreChat container — check after first chat interaction)"
docker compose -f ~/LibreChat/docker-compose.yml exec api npx --version && echo "npx: OK in container"
docker compose -f ~/LibreChat/docker-compose.yml exec api uvx --version 2>/dev/null && echo "uvx: OK in container" || echo "uvx: not available (SQLite/Git MCP may need manual install)"
```

> **Note on uvx in container:** LibreChat's official Docker image includes Node.js and npx but may not include Python/uvx. If uvx is missing, the SQLite and Git MCP servers will fail to spawn. See the LC_MCP_Core_Servers companion document for the workaround (thin Python wrapper scripts or npm-based alternatives).

---

## PHASE 5 — Model Router

**No changes.** The Phase 5 router is completely UI-independent. Follow Phase 5 of the M4 AI Stack Setup Guide v6.2 exactly. The router runs on `:8000`, `librechat.yaml` points to it as `http://host.docker.internal:8000/v1/`, and all virtual model routing, VRAM management, and fallback chains work identically.

---

## PHASE 6 — ComfyUI Integration (Action Required)

LibreChat does **not** have native ComfyUI integration. Open WebUI spoke to ComfyUI directly via a built-in plugin. LibreChat expects a DALL-E-compatible OpenAI image generation endpoint — ComfyUI does not expose one natively.

You have two paths:

### Option A — ComfyUI OpenAI Proxy (Recommended)

Run a lightweight proxy that translates LibreChat's DALL-E requests into ComfyUI workflow API calls. This makes image generation seamless in the LibreChat chat interface — the image appears inline, identical to how DALL-E behaves.

> ⚠️ **No single canonical package exists for this.** The comfyui-to-OpenAI proxy space is fragmented across multiple community projects. Do not attempt to install any package without first finding a maintained project on GitHub.

**Step 1 — Find a maintained proxy project:**

Search GitHub for: `comfyui openai proxy` or `comfyui dalle compatible`

Evaluate candidates by: last commit within 6 months, FLUX workflow support, active issues/PRs, Docker image available. As of early 2026 several viable projects exist; pick the one with the most recent activity.

**Step 2 — Clone and run (typical pattern):**

```bash
# Clone the project you chose
git clone https://github.com/<chosen-project>/comfyui-openai-proxy.git ~/comfyui-proxy
cd ~/comfyui-proxy

# Most projects support a Docker run — check the project's README for the exact command.
# Typical pattern:
docker build -t comfyui-proxy .
docker run -d --name comfyui-proxy \
  -p 127.0.0.1:8189:8189 \
  -e COMFYUI_URL=http://host.docker.internal:8188 \
  comfyui-proxy

# Alternatively, if the project publishes a Docker image:
# docker run -d --name comfyui-proxy \
#   -p 127.0.0.1:8189:8189 \
#   -e COMFYUI_URL=http://host.docker.internal:8188 \
#   ghcr.io/<chosen-project>/comfyui-proxy:latest
```

**Step 3 — Verify:**

```bash
curl -s http://localhost:8189/v1/models | python3 -m json.tool | head -5
# Expected: JSON list of model names (e.g., flux-schnell, flux-dev)
```

**Step 4 — Configure LibreChat:**

Add to `~/LibreChat/.env`:

```bash
echo "DALLE_API_KEY=comfyui-local" >> ~/LibreChat/.env
echo "DALLE_REVERSE_PROXY=http://host.docker.internal:8189/v1" >> ~/LibreChat/.env
```

Add to `~/LibreChat/librechat.yaml` under `endpoints:`:

```yaml
  openAI:
    apiKey: "comfyui-local"
    baseURL: "http://host.docker.internal:8189/v1"
    models:
      default: ["flux-schnell", "flux-dev", "sd35-large"]
      fetch: false
    titleConvo: false
    modelDisplayLabel: "ComfyUI Images"
```

Restart LibreChat: `docker compose -f ~/LibreChat/docker-compose.yml restart api`

### Option B — ComfyUI Agent with Action (No Proxy)

Build a LibreChat Agent that calls ComfyUI's native `/prompt` API directly via an OpenAPI Action. This is more explicit (user must chat with the "Image Generator" agent) but requires no additional software.

```yaml
# ComfyUI Action OpenAPI spec
openapi: 3.0.0
info:
  title: ComfyUI Image Generation
  description: Generate images using ComfyUI via FLUX or SD workflows
  version: "1.0"
servers:
  - url: http://host.docker.internal:8188
paths:
  /prompt:
    post:
      operationId: generateImage
      summary: Queue an image generation workflow
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [prompt]
              properties:
                prompt:
                  type: object
                  description: "ComfyUI workflow JSON — the agent constructs this"
      responses:
        "200":
          description: Prompt ID for status polling
  /history/{prompt_id}:
    get:
      operationId: getGenerationStatus
      summary: Check if generation is complete and get output filename
      parameters:
        - name: prompt_id
          in: path
          required: true
          schema:
            type: string
```

**Recommendation:** Use Option A if you want seamless inline image generation matching the Open WebUI experience. Use Option B if you prefer the agent-based approach consistent with the rest of the LibreChat tool strategy.

**Regardless of option chosen:** ComfyUI itself still runs exactly as documented in Phase 6 of the M4 AI Stack Setup Guide v6.2 — same install, same models, same `launch_comfyui.sh`. Only the LibreChat-side connection changes.

### ✅ Phase 6 Preflight Check

```bash
# ComfyUI itself (unchanged)
curl -s http://localhost:8188/system_stats | python3 -m json.tool | head -5

# If using Option A proxy:
# curl -s http://localhost:8189/v1/models | python3 -m json.tool

# If using Option B agent:
curl -s http://localhost:8188/object_info | python3 -m json.tool | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'ComfyUI nodes: {len(d)} available')"
```

---

## PHASE 7 — MusicGen (Music Generation)

**No changes.** Follow Phase 7 of the M4 AI Stack Setup Guide v6.2 exactly. The `music_api.py` runs natively on macOS, writes to `~/AI_Output/audio/`, and returns Caddy HTTP URLs. LibreChat accesses these through Agents and Actions rather than the Open WebUI tool system — see Phase 9 below.

---

## PHASE 8 — Voice Clone

**No changes.** Follow Phase 8 of the M4 AI Stack Setup Guide v6.2 exactly.

---

## PHASE 9 — Agents and Actions (LibreChat Tool Replacement)

This phase replaces Open WebUI's Python tool system. Open WebUI tools are Python scripts that run inside the OWUI container. LibreChat uses **Agents** (no-code workflow builders) and **Actions** (OpenAPI specifications) instead.

### What Open WebUI Tools Become in LibreChat

| Open WebUI Tool | LibreChat Equivalent |
|---|---|
| Generate Music (Phase 9.1) | Agent: "Music Generator" with MusicGen Action |
| Voice Clone (Phase 9.2) | Agent: "Voice Clone" with VoiceClone Action |
| Validate Splunk SPL (Phase 9.3) | Agent: "Splunk SecOps" with Splunk Action + auto-splunk model |
| Generate Presentation (Phase 9.4) | Agent: "Slide Deck Builder" with Presenton Action |
| Generate Document (Phase 11.6) | Agent: "Document Writer" with DocGen Action |

**Key difference:** In Open WebUI, tools are globally available and enabled per-workspace. In LibreChat, tools (Actions) are assigned to specific Agents, and you chat with the Agent directly. This is more explicit — you know exactly what tools are available for each agent.

### Creating Actions

Actions are OpenAPI specs that tell the Agent how to call your backend APIs. Create these in LibreChat's Agent Builder (click the robot icon → New Agent → Add Action).

#### MusicGen Action

```yaml
openapi: 3.0.0
info:
  title: MusicGen Music Generation
  description: Generate music and beats from text descriptions
  version: "1.0"
servers:
  - url: http://host.docker.internal:8001
paths:
  /generate:
    post:
      operationId: generateMusic
      summary: Generate music from a text prompt
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [prompt]
              properties:
                prompt:
                  type: string
                  description: "Text description of desired music"
                duration:
                  type: integer
                  default: 30
                  description: "Duration in seconds (5-60)"
                temperature:
                  type: number
                  default: 1.0
                  description: "Creativity 0.5-1.5"
      responses:
        "200":
          description: Generated audio file URL and metadata
  /health:
    get:
      operationId: checkMusicHealth
      summary: Check if MusicGen is running and model is loaded
```

#### DocGen Action

```yaml
openapi: 3.0.0
info:
  title: DocGen Document Generation
  description: Generate DOCX, PDF, and Markdown documents from markdown content
  version: "1.0"
servers:
  - url: http://host.docker.internal:8002
paths:
  /generate:
    post:
      operationId: generateDocument
      summary: Generate a document from markdown content with audit metadata
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [content]
              properties:
                content:
                  type: string
                  description: "Markdown content for the document"
                title:
                  type: string
                  default: "Document"
                author:
                  type: string
                  default: "OT Security Operations"
                project:
                  type: string
                  default: "LS-Power"
                formats:
                  type: array
                  items:
                    type: string
                    enum: [docx, pdf, md]
                  default: [docx, pdf]
                cip_requirements:
                  type: array
                  items:
                    type: string
                  default: []
                watermark:
                  type: string
                  default: ""
      responses:
        "200":
          description: Document URLs and metadata
  /recent-docs:
    get:
      operationId: listRecentDocs
      summary: List recently generated documents
      parameters:
        - name: n
          in: query
          schema:
            type: integer
            default: 10
  /health:
    get:
      operationId: checkDocGenHealth
      summary: Check DocGen, Pandoc, and Typst status
```

#### Voice Clone Action

```yaml
openapi: 3.0.0
info:
  title: Voice Clone TTS
  description: Text-to-speech with voice cloning via XTTS-v2
  version: "1.0"
servers:
  - url: http://host.docker.internal:5002
paths:
  /health:
    get:
      operationId: checkVoiceHealth
      summary: Check Voice Clone model status
```

> **Note on Voice Clone:** The `/clone` endpoint requires multipart file upload, which OpenAPI Actions in LibreChat handle differently from JSON endpoints. The health check Action is sufficient to verify status; for full synthesis, use the Filesystem MCP to reference pre-staged voice sample paths and call the API via the agent's reasoning.

#### Presenton Action

```yaml
openapi: 3.0.0
info:
  title: Presenton Presentation Generation
  description: Generate PPTX and PDF presentations from text descriptions
  version: "1.0"
servers:
  - url: http://host.docker.internal:5000
paths:
  /api/v1/ppt/presentation/generate:
    post:
      operationId: generatePresentation
      summary: Generate a presentation on any topic
      requestBody:
        required: true
        content:
          application/x-www-form-urlencoded:
            schema:
              type: object
              required: [content]
              properties:
                content:
                  type: string
                  description: "Topic or outline for the presentation"
                n_slides:
                  type: integer
                  default: 8
                tone:
                  type: string
                  default: "professional"
                  enum: [default, casual, professional, funny, educational, sales_pitch]
                verbosity:
                  type: string
                  default: "standard"
                  enum: [concise, standard, text-heavy]
                theme:
                  type: string
                  default: "professional"
                  enum: [classic, general, modern, professional]
                export_as:
                  type: string
                  default: "pptx"
                  enum: [pptx, pdf]
      responses:
        "200":
          description: Presentation metadata and file path
  /api/v1/ppt/health:
    get:
      operationId: checkPresentonHealth
      summary: Check if Presenton is running
```

### Pre-built Agent Recommendations

Create these agents in LibreChat's Agent Builder:

| Agent Name | Model Spec | MCP Tools | Actions | System Prompt Hint |
|---|---|---|---|---|
| General Assistant | `auto` | Filesystem, Memory, Time, Scrapling | — | General work, research, analysis |
| Code Assistant | `auto-coding` | Filesystem, Git, Scrapling | — | Scripting, code review, debugging |
| Agentic Coder | `auto-agentic-code` | Filesystem, Git, Sequential Thinking | — | Multi-file editing, codebase exploration |
| BigFix Expert | `auto-bigfix` | Filesystem, Scrapling | — | Fixlets, relevance, BES XML |
| Splunk SecOps | `auto-splunk` | Filesystem, SQLite, Scrapling | — | SPL, dashboards, correlation searches |
| Document Writer | `auto-docgen` | Filesystem, Memory | DocGen Action | Reports, memos, compliance artifacts |
| Music Generator | `auto-fast` | — | MusicGen Action | Background music, beats |
| Slide Deck Builder | `auto-slides` | Filesystem | Presenton Action | Presentations |
| Compliance Analyst | `auto-reasoning` | Filesystem, Memory, SQLite, Sequential Thinking | DocGen Action | NERC CIP audit prep |

---

## PHASE 10 — Document RAG (LibreChat RAG API)

LibreChat's RAG implementation uses a separate FastAPI service (`rag_api`) with a pgvector PostgreSQL backend instead of Open WebUI's built-in ChromaDB. This is more powerful but requires more setup.

### Option A — RAG API with pgvector (Full)

This is the full-featured path and is included in LibreChat's official Docker Compose. It was enabled via the `RAG_PORT` and `RAG_API_URL` entries in your `.env`.

```bash
# The RAG API and pgvector are included in LibreChat's base docker-compose.yml.
# If they're not starting, verify your .env has:
# RAG_PORT=8090
# RAG_API_URL=http://rag_api:8090

cd ~/LibreChat
docker compose up rag_api pgvector -d

curl -s http://localhost:8090/health
# Expected: {"status":"ok"}
```

To upload documents for RAG:
1. In LibreChat, click the **paperclip** icon in any chat
2. Upload PDFs, DOCX, or text files
3. The RAG API indexes them into pgvector automatically
4. Use `#filename` in chat to reference indexed documents

### Option B — Filesystem MCP (Lightweight Alternative)

If you primarily work with local files already organized in `~/Documents/` or `~/Projects/`, the Filesystem MCP server provides direct file access without indexing. The model reads files on-demand.

This is equivalent in practice for most compliance document work where files are few and well-organized. Use the RAG API when you have large document corpora (hundreds of files) that benefit from vector search.

### Policy Document Upload Strategy

For NERC CIP compliance documents:
- **RAG API:** Upload standard bodies, large policy collections, reference docs
- **Filesystem MCP:** Real-time access to working documents, evidence files, scripts
- **Memory MCP:** Store key facts, decisions, context that spans multiple documents

---

## PHASE 11 — DocGen API

**No changes.** The DocGen API (`docgen_api.py`) on `:8002` is completely unchanged. Follow Phase 11 of the M4 AI Stack Setup Guide v6.2 exactly. The Caddy `/docs/*` route is unchanged.

The only difference is how you call it: via the DocGen Action in a LibreChat Agent instead of via an Open WebUI tool.

---

## PHASE 11.8 — Presenton (Unchanged)

**No changes.** Presenton runs identically. Follow Phase 11.8 of the M4 AI Stack Setup Guide v6.2.

In LibreChat, access Presenton via the "Slide Deck Builder" agent (configured in Phase 9 above) or directly at `http://localhost:5000`.

---

## PHASE 12 — Master Launcher (Updated)

The launcher changes are minimal:
1. LibreChat replaces the Open WebUI Docker Compose section
2. The mcpo launch block is **removed** (LibreChat handles stdio MCP natively)
3. The Scrapling launch block remains

```bash
cat > ~/launch_ai_stack.sh << 'LAUNCHEOF'
#!/bin/zsh
# Usage:
#   ~/launch_ai_stack.sh                    — full stack (default, ~55GB)
#   AISTACK_MODE=core ~/launch_ai_stack.sh  — router + librechat only (~28GB)
#   AISTACK_MODE=minimal ~/launch_ai_stack.sh — ollama + router + caddy (~24GB)

GREEN='\033[0;32m'; YELLOW='\033[1;33m'; RED='\033[0;31m'; BLUE='\033[0;34m'; NC='\033[0m'
banner() { echo -e "${BLUE}$1${NC}"; }
ok()     { echo -e "${GREEN}✓ $1${NC}"; }
warn()   { echo -e "${YELLOW}⚡ $1${NC}"; }
fail()   { echo -e "${RED}✗ $1${NC}"; }

MODE=${AISTACK_MODE:-"full"}
case "$MODE" in
  core)     MODE_LABEL="Core (~28GB)"    ;;
  minimal)  MODE_LABEL="Minimal (~24GB)" ;;
  personal) MODE_LABEL="Personal (~40GB)" ;;
  *)        MODE_LABEL="Full (~55GB)"    ;;
esac

banner "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
banner "   M4 Mini AI Stack v1.0 (LibreChat) — ${MODE_LABEL}"
banner "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

if [[ "$MODE" == "personal" ]]; then
    export PERSONAL_MODE=1
fi

# ── 1. Ollama ────────────────────────────────────────────────────
launchctl setenv OLLAMA_FLASH_ATTENTION 1
launchctl setenv OLLAMA_KV_CACHE_TYPE q8_0
if ! pgrep -x "ollama" > /dev/null; then
  /opt/homebrew/bin/brew services start ollama
  sleep 4
fi
ok "Ollama          → localhost:11434"

# ── 2. Router ────────────────────────────────────────────────────
if ! lsof -i 127.0.0.1:8000 > /dev/null 2>&1; then
  ~/launch_router.sh
  if [ $? -ne 0 ]; then
    fail "Router failed to start. Aborting."
    exit 1
  fi
fi
ok "Model Router    → localhost:8000"

# ── 2a. Router Watchdog ──────────────────────────────────────────
WATCHDOG_RUNNING=false
if [ -f /tmp/watchdog.pid ]; then
  WD_PID=$(cat /tmp/watchdog.pid)
  if kill -0 "$WD_PID" 2>/dev/null; then
    ok "Router Watchdog → already running (PID $WD_PID)"
    WATCHDOG_RUNNING=true
  fi
fi
if ! $WATCHDOG_RUNNING; then
(
  while true; do
    sleep 30
    if ! curl -s http://localhost:8000/health > /dev/null 2>&1; then
      if [ -f /tmp/router.pid ]; then
        RPID=$(cat /tmp/router.pid)
        if kill -0 "$RPID" 2>/dev/null; then continue; fi
      fi
      echo "[Watchdog $(date '+%H:%M:%S')] Router dead — restarting..."
      ~/launch_router.sh
    fi
  done
) &
echo $! > /tmp/watchdog.pid
ok "Router Watchdog → started (PID $(cat /tmp/watchdog.pid))"
fi

# ── 2b. Warm-cache ───────────────────────────────────────────────
(sleep 10 && curl -s -X POST http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3:8b","keep_alive":-1,"prompt":"warmup"}' > /dev/null) &
(curl -s -X POST http://localhost:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3:0.6b","keep_alive":-1,"prompt":"classify"}' > /dev/null) &
ok "Warm-cache      → qwen3:8b + qwen3:0.6b loading in background"

# ── 3. Caddy ─────────────────────────────────────────────────────
if ! caddy list 2>/dev/null | grep -q "8080"; then
  caddy start --config ~/ai-stack/caddy/Caddyfile --adapter caddyfile
fi
ok "Caddy           → http://localhost:8080"

# ── 4. Docker: LibreChat + dependencies ──────────────────────────
if [[ "$MODE" == "minimal" ]]; then
  warn "Docker          → skipped (minimal mode)"
  warn "LibreChat       → NOT available in minimal mode"
else
  docker_ready=false
  if ! docker info > /dev/null 2>&1; then
    warn "Docker not running — launching Docker Desktop..."
    open /Applications/Docker.app
    for i in $(seq 1 24); do
      sleep 5
      if docker info > /dev/null 2>&1; then docker_ready=true; break; fi
      echo "  Waiting for Docker... (${i}/24 × 5s)"
    done
  else
    docker_ready=true
  fi

  if $docker_ready; then
    # SearxNG — use standalone compose for fresh installs; fall back to M4 stack compose for migrants
    cd ~/ai-stack
    if [ -f ~/ai-stack/docker-compose-searxng.yml ]; then
      docker compose -f docker-compose-searxng.yml up -d
    else
      docker compose up searxng -d 2>/dev/null || true  # M4 stack migrant fallback
    fi

    # LibreChat stack
    cd ~/LibreChat
    docker compose up -d
    sleep 8

    ok "LibreChat       → localhost:3080 (via Caddy: localhost:8080)"
    ok "MongoDB         → localhost:27017"
    ok "Meilisearch     → localhost:7700"
    ok "Redis           → localhost:6379"
    ok "SearxNG         → localhost:8888"
  else
    fail "Docker did not start after 2 minutes."
  fi
fi

# ── 5. ComfyUI (full mode only) ──────────────────────────────────
if [[ "$MODE" == "full" ]]; then
  if ! lsof -i 127.0.0.1:8188 > /dev/null 2>&1; then
    ~/launch_comfyui.sh
    echo -n "  Waiting for ComfyUI..."
    for i in $(seq 1 20); do
      sleep 3
      if curl -s http://localhost:8188/system_stats > /dev/null 2>&1; then
        echo " ready"; break
      fi
      echo -n "."
      [ $i -eq 20 ] && echo " timeout — check manually"
    done
  fi
  ok "ComfyUI         → localhost:8188"
else
  warn "ComfyUI         → skipped (${MODE} mode). Run ~/launch_comfyui.sh when needed."
fi

warn "MusicGen        → ~/launch_musicgen.sh (on-demand)"
warn "Voice Clone     → ~/launch_voiceclone.sh (on-demand)"
warn "DocGen          → ~/launch_docgen.sh (on-demand)"

# ── 6. Scrapling MCP (auto-start if installed) ───────────────────
# mcpo is NOT started — LibreChat handles stdio MCP servers natively.
if [ -f ~/launch_scrapling_mcp.sh ]; then
    if ! lsof -i :8900 > /dev/null 2>&1; then
        ~/launch_scrapling_mcp.sh
    fi
    ok "Scrapling MCP    → localhost:8900"
else
    warn "Scrapling MCP   → skipped (not installed — see LC_MCP_Scrapling.md)"
fi

# ── 7. Personal mode banner ───────────────────────────────────────
if [[ "$MODE" == "personal" ]]; then
    banner ""
    banner "🔓 Personal mode — uncensored models active"
    banner "   Reasoning:  auto-no-filter (DeepSeek-R1-Abliterated 32B)"
    banner "   Fast:       auto-no-filter-fast (Dolphin 3.0 8B)"
    banner "   Cybersec:   auto-cybersec (WhiteRabbitNeo 33B)"
    banner "   Creative:   auto-nsfw (Hermes 3 Lorablated 8B)"
fi

banner ""
banner "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
banner "  Stack UP [${MODE_LABEL}] → http://localhost:8080"
banner "  Routing  → http://localhost:8080/router/routing-info"
banner "  Telemetry→ http://localhost:8080/router/telemetry"
banner "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
LAUNCHEOF
chmod +x ~/launch_ai_stack.sh
```

**LaunchAgent update** — Update the version string in `~/Library/LaunchAgents/com.aistack.launcher.plist` if desired, but the plist itself does not need changes — it simply calls `launch_ai_stack.sh`.

---

## PHASE 13 — Smoke Test (Updated for LibreChat)

```bash
cat > ~/smoke_test_lc.sh << 'SMOKEEOF'
#!/bin/bash
PASS=0; FAIL=0

check() {
  local name="$1"; local cmd="$2"
  if eval "$cmd" > /dev/null 2>&1; then
    echo "✅ $name"; ((PASS++))
  else
    echo "❌ $name"; ((FAIL++))
  fi
}

echo ""
echo "══════════ LibreChat AI Stack v1.0 Smoke Test ══════════"

echo ""
echo "── Core services ──"
check "Ollama"            "curl -sf http://localhost:11434/api/tags"
check "Model Router"      "curl -sf http://localhost:8000/health"
check "Caddy"             "curl -sf http://localhost:8080 -o /dev/null"
check "LibreChat"         "curl -sf http://localhost:3080 -o /dev/null"
check "MongoDB"           "docker compose -f ~/LibreChat/docker-compose.yml exec mongodb mongosh --eval 'db.runCommand({ping:1})' --quiet 2>/dev/null | grep -q ok"
check "Meilisearch"       "curl -sf http://localhost:7700/health | python3 -c \"import sys,json; exit(0 if json.load(sys.stdin)['status']=='available' else 1)\""
check "Redis"             "docker compose -f ~/LibreChat/docker-compose.yml exec redis redis-cli ping 2>/dev/null | grep -q PONG"
check "SearxNG JSON"      "curl -sf 'http://localhost:8888/search?q=test&format=json' | python3 -c \"import sys,json; exit(0 if len(json.load(sys.stdin).get('results',[])) > 0 else 1)\""
check "SearxNG from container" "docker compose -f ~/LibreChat/docker-compose.yml exec api curl -sf 'http://host.docker.internal:8888/search?q=test&format=json' | python3 -c \"import sys,json; exit(0 if len(json.load(sys.stdin).get('results',[])) > 0 else 1)\""
check "ComfyUI"           "curl -sf http://localhost:8188/system_stats"

echo ""
echo "── Caddy file serving ──"
echo "caddy-test" > ~/AI_Output/audio/caddy_smoke_test.txt
check "Caddy /audio/"     "curl -sf http://localhost:8080/audio/caddy_smoke_test.txt | grep -q caddy-test"
check "Router via Caddy"  "curl -sf http://localhost:8080/router/health | python3 -m json.tool | grep -q ok"
rm -f ~/AI_Output/audio/caddy_smoke_test.txt

echo ""
echo "── Models ──"
check "qwen3:32b"           "ollama list | grep -q 'qwen3:32b'"
check "bigfix-expert"       "ollama list | grep -q bigfix-expert"
check "splunk-secops"       "ollama list | grep -q splunk-secops"
check "Virtual 'auto'"      "curl -sf http://localhost:8000/api/tags | python3 -m json.tool | grep -q '\"auto\"'"
check "Virtual 'auto-bigfix'" "curl -sf http://localhost:8000/api/tags | python3 -m json.tool | grep -q '\"auto-bigfix\"'"

echo ""
echo "── Routing logic ──"
get_routed_model() {
  local prompt="$1"
  timeout 10s curl -s -X POST http://localhost:8000/api/chat \
    -H "Content-Type: application/json" \
    -d "$(python3 -c "import json; print(json.dumps({'model':'auto','messages':[{'role':'user','content':'''$prompt'''}]}))")" \
  | python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if not line: continue
    try:
        d = json.loads(line)
        if 'model' in d:
            print(d['model'])
            break
    except: continue
"
}

BF_MODEL=$(get_routed_model "Write a BigFix Fixlet for CIP-007 patch management")
[[ "$BF_MODEL" == "bigfix-expert" ]] \
  && { echo "✅ BigFix routing → $BF_MODEL"; ((PASS++)); } \
  || { echo "❌ BigFix routing (got: '$BF_MODEL')"; ((FAIL++)); }

SP_MODEL=$(get_routed_model "Write a Splunk query with index= to detect failed logins")
[[ "$SP_MODEL" == "splunk-secops" ]] \
  && { echo "✅ Splunk routing → $SP_MODEL"; ((PASS++)); } \
  || { echo "❌ Splunk routing (got: '$SP_MODEL')"; ((FAIL++)); }

echo ""
echo "── Filesystem ──"
check "AI_Output/audio writable"  "touch ~/AI_Output/audio/.w && rm ~/AI_Output/audio/.w"
check "AI_Output/voice writable"  "touch ~/AI_Output/voice/.w && rm ~/AI_Output/voice/.w"
check "AI_Output/images writable" "touch ~/AI_Output/images/.w && rm ~/AI_Output/images/.w"
check "AI_Output/docs writable"   "touch ~/AI_Output/docs/.w && rm ~/AI_Output/docs/.w"
check "AI_Output/slides writable" "touch ~/AI_Output/slides/.w && rm ~/AI_Output/slides/.w"
check "LibreChat MCP memory dir"  "[ -d ~/LibreChat/data/mcp-memory ]"
check "LibreChat MCP sqlite dir"  "[ -d ~/LibreChat/data/mcp-sqlite ]"

echo ""
echo "── MCP Servers (Scrapling only — stdio via LibreChat container) ──"
if curl -sf http://localhost:8900/mcp -o /dev/null 2>&1; then
  check "Scrapling MCP" "curl -s -o /dev/null -w '%{http_code}' http://localhost:8900/mcp | grep -q 200"
else
  echo "⏭  Scrapling MCP not running (~/launch_scrapling_mcp.sh to start)"
fi

echo ""
echo "── On-demand services ──"
if curl -sf http://localhost:8001/health > /dev/null 2>&1; then
  check "MusicGen health"    "curl -sf http://localhost:8001/health | python3 -m json.tool | grep -q ok"
else
  echo "⏭  MusicGen not running"
fi
if curl -sf http://localhost:5002/health > /dev/null 2>&1; then
  check "Voice Clone health" "curl -sf http://localhost:5002/health | python3 -m json.tool | grep -q ok"
else
  echo "⏭  Voice Clone not running"
fi
if curl -sf http://localhost:8002/health > /dev/null 2>&1; then
  check "DocGen health"      "curl -sf http://localhost:8002/health | python3 -m json.tool | grep -q ok"
else
  echo "⏭  DocGen not running"
fi

echo ""
echo "════════════════════════════════════════════"
echo "Results: ${PASS} passed  ${FAIL} failed"
[[ $FAIL -eq 0 ]] \
  && echo "🎉 LibreChat stack operational → http://localhost:8080" \
  || echo "⚠  Fix the ❌ items above and re-run: bash ~/smoke_test_lc.sh"
echo ""
SMOKEEOF
chmod +x ~/smoke_test_lc.sh
```

---

## PHASE 14 — Personal Mode

**No changes.** The Personal Mode models are loaded by the Phase 5 router (not the UI), so the uncensored workspace routing, VRAM management, and personal model catalog are identical. Follow Phase 14 of the M4 AI Stack Setup Guide v6.2 exactly.

In LibreChat, personal mode workspaces appear in the model spec dropdown (if you added them to `librechat.yaml`) or can be accessed via the "Ollama Direct" endpoint with `@model:` syntax in chat.

To add personal mode model specs to `librechat.yaml`, append to the `modelSpecs.list` section:

```yaml
    - name: "auto-no-filter"
      label: "No Filter (Heavy)"
      description: "DeepSeek-R1-Abliterated 32B — personal mode only"
      preset:
        endpoint: "M4 Stack"
        model: "auto-no-filter"

    - name: "auto-cybersec"
      label: "Cybersec Lab"
      description: "WhiteRabbitNeo 33B — personal mode only"
      preset:
        endpoint: "M4 Stack"
        model: "auto-cybersec"
```

---

## PHASE 15 — MCP Servers (Architectural Change)

In the LibreChat stack, **there is no mcpo proxy**. LibreChat spawns stdio MCP servers directly as child processes inside its API container. This eliminates the `:9000` port, the mcpo process, and ~235MB of process overhead.

### Architecture: Native stdio + External HTTP + Remote Endpoints

| Component | Open WebUI Stack | LibreChat Stack |
|---|---|---|
| mcpo proxy `:9000` | Required | **Eliminated** |
| Filesystem MCP | Spawned by mcpo | Spawned by LibreChat API container |
| Memory MCP | Spawned by mcpo | Spawned by LibreChat API container |
| SQLite MCP | Spawned by mcpo | Spawned by LibreChat API container |
| Git MCP | Spawned by mcpo | Spawned by LibreChat API container |
| Sequential Thinking | Spawned by mcpo | Spawned by LibreChat API container |
| Time MCP | Spawned by mcpo | Spawned by LibreChat API container |
| Scrapling `:8900` | Native Streamable HTTP | Native Streamable HTTP (unchanged) |
| Microsoft Learn | Native Streamable HTTP | Native Streamable HTTP (unchanged) |

### MCP Tool Access in LibreChat

MCP tools appear in two places:

1. **In chat** — when chatting with a non-agent endpoint, a Tools dropdown below the input shows available MCP servers. Select one or more to enable their tools for that conversation.

2. **In Agent Builder** — when creating agents, assign specific MCP servers and enable/disable individual tools per agent. This gives per-agent tool control equivalent to Open WebUI's per-workspace tool assignments.

> ⚠️ **First-launch latency:** After container start, LibreChat takes 20-60 seconds to spawn all MCP server child processes and confirm their tool lists. During this window, the tools dropdown may show no MCP tools. Wait 30-60 seconds, then refresh the browser. This only happens at container startup — tools are available immediately in subsequent sessions once the container is warm.

> ⚠️ **Scrapling SSRF awareness:** Because Scrapling runs locally and the LLM controls what URLs it fetches, a prompt-injected or adversarial instruction could ask Scrapling to fetch internal endpoints (e.g., `http://localhost:8000/admin` or `http://host.docker.internal:27017`). Scrapling has no egress filtering. This is an accepted risk for a single-user local deployment, but be aware of it when using Scrapling with documents sourced from untrusted external content.

To restrict an MCP server to Agent Builder only (hide from chat dropdown):

```yaml
mcpServers:
  sequential-thinking:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-sequential-thinking"]
    chatMenu: false  # Only available in Agent Builder, not chat dropdown
```

### Companion Documents

| Document | What It Covers |
|---|---|
| `LC_MCP_Scrapling_v1.0.md` | Scrapling setup for LibreChat — unchanged process, updated context |
| `LC_MCP_Core_Servers_v1.0.md` | Native stdio MCP configuration in `librechat.yaml` |

---

## Memory Budget

| Component | Memory |
|---|---|
| macOS base | ~7GB |
| Ollama: qwen3:32b + qwen3:8b + qwen3:0.6b warm | ~26GB |
| ComfyUI + FLUX.1-Dev (when active) | ~24GB |
| LibreChat API container | ~800MB |
| MongoDB | ~300MB |
| Redis | ~50MB |
| Meilisearch | ~200MB |
| SearxNG | ~150MB |
| Caddy + Router | ~500MB |
| MusicGen or Voice Clone (on-demand) | ~4–7GB |
| **Safe headroom** | **~2–7GB** |

**Comparison to Open WebUI stack:** The LibreChat stack uses approximately 300MB less memory total. The primary savings come from the lighter UI container (~800MB vs ~1.5GB) and eliminating mcpo. MongoDB and Meilisearch add back some overhead, but the net result is favorable.

---

## Migration from Open WebUI

If you have an existing Open WebUI installation and are migrating to LibreChat:

- [ ] Export important conversations from Open WebUI (Settings → Export)
- [ ] Back up ChromaDB RAG data if you have indexed documents
- [ ] Back up `~/.mcp-memory/memory.json` (Memory MCP knowledge graph)
- [ ] Back up `~/data/ops.db` (SQLite MCP database)
- [ ] Stop Open WebUI: `docker compose -f ~/ai-stack/docker-compose.yml down`
- [ ] Update Caddy upstream from `:3000` to `:3080`
- [ ] Clone and configure LibreChat per Phase 4 above
- [ ] Update `launch_ai_stack.sh` per Phase 12 above
- [ ] Remove mcpo from any custom launcher scripts — no longer needed
- [ ] Copy `~/.mcp-memory/memory.json` to `~/LibreChat/data/mcp-memory/memory.json`
- [ ] Copy `~/data/ops.db` to `~/LibreChat/data/mcp-sqlite/ops.db`
- [ ] Start LibreChat: `cd ~/LibreChat && docker compose up -d`
- [ ] Register your account at `http://localhost:8080`
- [ ] Create Agents for tool workflows (Phase 9 above)
- [ ] Run smoke test: `bash ~/smoke_test_lc.sh`

**Optional: Remove Open WebUI after confirming LibreChat is stable**

Once you've run the smoke test and verified LibreChat is working, clean up the old Open WebUI containers and volumes to recover ~1.5GB of memory. Leaving them running wastes RAM; leaving them stopped but present risks accidental restart if you ever `docker compose up -d` in `~/ai-stack/` for any reason.

```bash
# Stop and remove Open WebUI containers
cd ~/ai-stack
docker compose down

# Remove volumes (conversations and ChromaDB vectors — ensure you've exported anything you need)
docker volume rm ai-stack_open_webui_data ai-stack_chromadb_data

# Verify volumes are gone
docker volume ls | grep ai-stack

# Keep ~/ai-stack/ for SearxNG (it still runs from docker-compose-searxng.yml)
# Keep ~/ai-stack/caddy/ (Caddy config is unchanged)
```

> ⚠️ `docker volume rm` is permanent. Confirm you've exported any Open WebUI conversations you want to keep before running this. The ChromaDB RAG data (indexed documents) is also deleted — re-index your documents via LibreChat's RAG API if needed.

---

## Troubleshooting

**LibreChat not reachable at :3080**
```bash
cd ~/LibreChat
docker compose logs api --tail=30
# Common: MongoDB not ready yet — LibreChat waits for it, check mongodb logs too
docker compose logs mongodb --tail=20
```

**MCP servers not appearing in chat**
```bash
# Restart LibreChat after any librechat.yaml changes
docker compose restart api

# Check MCP server startup in logs
docker compose logs api | grep -i mcp

# Verify npm/npx is available inside the container
docker compose exec api npx --version
```

**Router connection error in LibreChat**
```bash
# From inside the container — verify host.docker.internal resolves
docker compose exec api curl http://host.docker.internal:8000/health
# If it fails, ensure extra_hosts is in docker-compose.override.yml
```

**Meilisearch search not working**
```bash
curl http://localhost:7700/health
# If unavailable: docker compose restart meilisearch
# After restart, flush Redis:
docker compose exec redis redis-cli FLUSHALL
```

**SearxNG web search prompts for reconfiguration**
```bash
# Verify JSON format enabled in settings.yml, then flush Redis cache
docker compose -f ~/LibreChat/docker-compose.yml exec redis redis-cli FLUSHALL
# Verify reachable from inside LibreChat container:
docker compose -f ~/LibreChat/docker-compose.yml exec api \
  curl "http://host.docker.internal:8888/search?q=test&format=json" | python3 -m json.tool | head -5
```

**Filesystem MCP "path not allowed"**
```bash
# Container-side paths in librechat.yaml must match mounts in docker-compose.override.yml
# Verify the mount exists:
docker compose exec api ls /host-home/Projects
```

---

## Where Data Lives (LibreChat Edition)

| Data | Location |
|---|---|
| LibreChat conversations, users, agents | Docker volume `librechat_mongodb_data` |
| LibreChat session cache | Docker volume `librechat_redis_data` |
| Meilisearch search index | Docker volume `librechat_meilisearch_data` |
| RAG vectors (if RAG API enabled) | Docker volume `librechat_pgvector_data` |

> **Volume name note:** Docker Compose prefixes volume names with the project directory name. If you cloned LibreChat into `~/LibreChat/`, volumes are named `librechat_mongodb_data`, etc. If you used a different directory name, substitute accordingly. Verify: `docker volume ls | grep librechat`
| SearxNG config | `~/ai-stack/searxng/` |
| Caddy config | `~/ai-stack/caddy/Caddyfile` |
| LibreChat config | `~/LibreChat/librechat.yaml`, `~/LibreChat/.env` |
| MCP memory graph | `~/LibreChat/data/mcp-memory/memory.json` |
| MCP SQLite database | `~/LibreChat/data/mcp-sqlite/ops.db` |
| Ollama models | `~/.ollama/models/` |
| Generated images, audio, voice, docs | `~/AI_Output/` |

**Backups:**

```bash
mkdir -p ~/Backups

# LibreChat MongoDB conversations + users
docker run --rm -v librechat_mongodb_data:/data -v ~/Backups:/b \
  alpine tar czf /b/librechat_mongo_$(date +%Y%m%d).tar.gz /data

# MCP persistent data
cp ~/LibreChat/data/mcp-memory/memory.json \
   ~/Backups/mcp-memory-$(date +%Y%m%d).json
cp ~/LibreChat/data/mcp-sqlite/ops.db \
   ~/Backups/mcp-sqlite-$(date +%Y%m%d).db

# AI Output
tar czf ~/Backups/ai_output_$(date +%Y%m%d).tar.gz ~/AI_Output/
```

---

### Document Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0 | Feb 24, 2026 | Initial release — complete LibreChat stack guide covering all phases, architecture, migration path, smoke test, and memory budget. Companion to M4 AI Stack Setup Guide v6.2. |
| 1.1 | Feb 24, 2026 | **FIX (Critical):** Added standalone `docker-compose-searxng.yml` for fresh installs — SearxNG was silently never starting without a prior M4 stack. Launcher updated with compose file detection and M4 migrant fallback. **FIX (High):** `.env` generation changed to unquoted heredoc with inline `$(openssl rand ...)` expansion — eliminates literal placeholder strings that produced weak/non-functional JWT secrets. **FIX (High):** `docker-compose.override.yml` now uses `${HOST_HOME}` sourced from `.env` instead of `${HOME}` shell variable — reliable across manual launch and LaunchAgent contexts. npm cache volume added to prevent npx cold-start latency on MCP server spawn. **FIX (High):** SQLite initialization changed from `touch` (0-byte file) to `sqlite3 ... "SELECT 1;"` — prevents `"file is not a database"` error on first read query. **FIX (New):** Phase 6 ComfyUI correctly documented as requiring integration work — provides Option A (OpenAI proxy) and Option B (ComfyUI Action) paths with implementation guidance. **FIX:** Phase 15 heading changed from inaccurate "Two Connections" to "Native stdio + External HTTP + Remote Endpoints". **ADD:** Scrapling SSRF risk documented with single-user acceptability note. **ADD:** First-launch MCP tool latency (20-60s) documented in tool access section. **ADD:** Docker volume naming caveat in "Where Data Lives". |
| 1.2 | Feb 24, 2026 | **FIX (High):** Phase 6 Option A pip install command removed — `comfyui-openai-proxy` does not exist on PyPI; replaced with GitHub-search-first approach and generic Docker build/run pattern with step-by-step verification. **FIX (High):** Added `modelSpecs` vs router routing explanation after `librechat.yaml` heredoc — explains that LibreChat specs are UI presets while Phase 5 router does actual routing enforcement; clarifies `enforce: false` behavior for Open WebUI migrants. **FIX (Medium):** `fetch: false` in `librechat.yaml` expanded to 5-line comment explicitly warning against changing it to `true` — prevents workspace routing breakage from future troubleshooting attempts. **FIX (Medium):** Migration checklist extended with optional Open WebUI cleanup block (`docker compose down` + `docker volume rm`) and permanence warning — prevents ~1.5GB memory waste from orphaned containers. **FIX (Medium):** Smoke test now includes container-side SearxNG reachability check (`docker compose exec api curl ... host.docker.internal:8888`) alongside host-side check — proves Docker bridge network is functional. |

---

*LibreChat_Stack_Setup_Guide.md v1.2 — LibreChat Edition of M4 AI Stack · Companion to v6.2*
