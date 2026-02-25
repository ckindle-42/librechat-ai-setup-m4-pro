# MCP Core Servers: Native stdio via LibreChat
### Companion document to LibreChat Stack Setup Guide v1.1 — Phase 15
**Document version:** 1.1 (Feb 24, 2026)

**Architecture:** Native stdio spawn (LibreChat API container)  
**Config:** `~/LibreChat/librechat.yaml` — `mcpServers` block  
**Servers hosted:** 6 (Filesystem, Memory, SQLite, Git, Sequential Thinking, Time)  
**Plus remote:** Microsoft Learn (streamable-http, zero local install)  
**Total idle memory:** ~235MB (same processes, different parent — LibreChat vs mcpo)  

---

## What This Does (and What Changed)

In the Open WebUI stack, stdio-based MCP servers could not connect directly to Open WebUI — they required the `mcpo` proxy (port `:9000`) to convert them to OpenAPI HTTP endpoints. mcpo was a necessary translation layer.

**LibreChat eliminates this entirely.** LibreChat's API container spawns stdio MCP servers directly as child processes, manages their lifecycle, and exposes their tools natively. The same six servers run — Filesystem, Memory, SQLite, Git, Sequential Thinking, and Time — but they are now children of the LibreChat process instead of children of mcpo.

**What you gain:**
- `:9000` port eliminated
- `~/launch_mcpo.sh` no longer needed
- No `mcpo` process, no JSON config for mcpo, no OpenAPI conversion overhead
- Simpler architecture — the MCP tools just work from day one

**What you lose:**
- The mcpo Swagger UI at `http://localhost:9000/docs` for debugging individual tools
- The ability to add new MCP servers without restarting LibreChat (mcpo could hot-reload config)

---

## Prerequisites

- **LibreChat running** per Phase 4 of the LibreChat Stack Setup Guide
- **Node.js 18+ and npx** available inside the LibreChat API container (included in official image)
- **Python + uvx** availability inside the container for SQLite and Git servers (see note below)

### Node.js / npm in Container

LibreChat's official Docker image (`ghcr.io/danny-avila/librechat`) includes Node.js, npm, and npx. The Filesystem, Memory, Sequential Thinking, and Time servers (all `@modelcontextprotocol/*` npm packages) work out of the box.

Verify:
```bash
docker compose -f ~/LibreChat/docker-compose.yml exec api npx --version
# Expected: version number (e.g. 10.x.x)
```

### Python / uvx in Container

`mcp-server-sqlite` and `mcp-server-git` use `uvx` (Python). The LibreChat API container may not include Python or uvx. Check:

```bash
docker compose -f ~/LibreChat/docker-compose.yml exec api uvx --version 2>/dev/null \
  && echo "uvx: available" || echo "uvx: NOT available — see workaround below"
```

**If uvx is not available**, use the npm-based alternatives documented in the Per-Server sections below. Both SQLite and Git have npm-compatible equivalents.

---

## Configuration

All MCP server configuration lives in the `mcpServers` block of `~/LibreChat/librechat.yaml`. This block is already present if you followed Phase 4 of the LibreChat Stack Setup Guide. This document provides detailed reference for each server.

After any change to `librechat.yaml`, restart the LibreChat API container:
```bash
docker compose -f ~/LibreChat/docker-compose.yml restart api
# MCP servers restart with it — they are child processes
```

---

## Filesystem MCP Server

**What:** Secure read/write access to directories mounted into the LibreChat container. The model can browse, read, write, search, and manage files — but ONLY within the allowed paths.

### Configuration in `librechat.yaml`

```yaml
mcpServers:
  filesystem:
    title: "Filesystem"
    description: "Read/write/search local project files, scripts, compliance docs"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-filesystem"
      - "/host-home/Projects"
      - "/host-home/Scripts"
      - "/host-home/Documents"
      - "/host-home/AI_Output"
```

### Container-Side Mount Points

The paths in `args` are **container-side paths** — they must match the bind mounts in `docker-compose.override.yml`:

```yaml
# In docker-compose.override.yml:
services:
  api:
    volumes:
      - type: bind
        source: ${HOME}/Projects      # Mac filesystem
        target: /host-home/Projects   # Container path (matches librechat.yaml)
      - type: bind
        source: ${HOME}/Scripts
        target: /host-home/Scripts
      - type: bind
        source: ${HOME}/Documents
        target: /host-home/Documents
      - type: bind
        source: ${HOME}/AI_Output
        target: /host-home/AI_Output
```

If you add a new directory to `librechat.yaml` args, you must also add the corresponding bind mount to `docker-compose.override.yml` and restart with `docker compose up -d` (not just `restart api`).

### Available Tools

| Tool | Description |
|---|---|
| `read_file` | Read complete contents of a file |
| `read_multiple_files` | Read several files at once |
| `write_file` | Create or overwrite a file |
| `create_directory` | Create a new directory |
| `list_directory` | List files and subdirectories |
| `move_file` | Move or rename files/directories |
| `search_files` | Recursive search by filename pattern |
| `get_file_info` | File metadata (size, modified date, permissions) |
| `list_allowed_directories` | Show which directories are accessible |

### Example Prompts

```
"Read the file at /host-home/Scripts/bigfix/cip007_patch.sh and explain what it does"

"Search /host-home/Projects for any Python file that imports tenable"

"Write this remediation script to /host-home/Scripts/ir-automation/collect_logs.py"

"List everything in /host-home/Documents/compliance/ modified in the last 30 days"
```

> **Important:** Use container-side paths in prompts — `/host-home/Projects/...` not `~/Projects/...`. The model is running in a context where the Mac filesystem is accessible only via the container mounts.

### Troubleshooting

```bash
# Verify mount exists inside container
docker compose exec api ls /host-home/Projects

# If "path not allowed" error:
# 1. Check args in librechat.yaml match the target paths in docker-compose.override.yml
# 2. Restart stack: docker compose up -d (needed for volume changes, not just restart api)
```

---

## Memory MCP Server

**What:** A persistent knowledge graph that stores entities, relationships, and observations in a local JSON file. Survives across chat sessions — gives your local LLM persistent memory about your environment.

### Configuration in `librechat.yaml`

```yaml
mcpServers:
  memory:
    title: "Memory"
    description: "Persistent knowledge graph — remember environment details across sessions"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-memory"
    env:
      MEMORY_FILE_PATH: "/app/data/mcp-memory/memory.json"
```

The `/app/data/mcp-memory` directory is mounted from `~/LibreChat/data/mcp-memory/` via `docker-compose.override.yml`. The JSON file is created on the first `create_entities` call.

### Available Tools

| Tool | Description |
|---|---|
| `create_entities` | Add new entities (servers, projects, team members, decisions) |
| `create_relations` | Define relationships between entities |
| `add_observations` | Attach facts/observations to entities |
| `delete_entities` | Remove entities from the graph |
| `delete_observations` | Remove specific observations |
| `delete_relations` | Remove relationships |
| `read_graph` | Read the entire knowledge graph |
| `search_nodes` | Search for entities by name or content |
| `open_nodes` | Retrieve specific entities by name |

### Example Prompts

```
"Remember that the Texas region uses BigFix server TX-BIGFIX-01 at 10.1.2.50,
 managed by Jake on my team"

"What do you know about our Splunk deployment?"

"Remember that we decided to use Tenable agent scanning instead of network scanning
 for the California DMZ. Change was approved by [manager] on Feb 20."

"Add an entity for the CIP-007 R2 audit — due date March 15, covers Texas and
 New York, primary evidence owner is Sarah"

"Show me the full knowledge graph"
```

### System Prompt Tip

Add to the system prompt of any agent where persistent memory matters:

```
You have access to a persistent Knowledge Graph memory system.
Use it to remember important details about the user's environment,
team members, infrastructure, projects, and decisions.
When the user shares important facts, store them as entities with
relationships. When asked about past decisions or configurations,
search your memory first.
```

### Data Location and Backup

```bash
# Memory graph location on Mac
ls -la ~/LibreChat/data/mcp-memory/memory.json

# Backup
cp ~/LibreChat/data/mcp-memory/memory.json \
   ~/LibreChat/data/mcp-memory/memory.json.bak.$(date +%Y%m%d)

# The file is plain JSON — inspect with:
cat ~/LibreChat/data/mcp-memory/memory.json | python3 -m json.tool | head -50
```

> **Migration from mcpo:** If you were using the Memory MCP via mcpo in the Open WebUI stack, your knowledge graph is at `~/.mcp-memory/memory.json`. Copy it to the new location:
> ```bash
> cp ~/.mcp-memory/memory.json ~/LibreChat/data/mcp-memory/memory.json
> ```

---

## SQLite MCP Server

**What:** Full SQL access to a local SQLite database from chat. Create tables, insert records, query data, and generate insights — all from natural language.

### Configuration in `librechat.yaml`

```yaml
mcpServers:
  sqlite:
    title: "SQLite"
    description: "Query local databases — findings tracker, asset inventory, analytics"
    command: "uvx"
    args:
      - "mcp-server-sqlite"
      - "--db-path"
      - "/app/data/mcp-sqlite/ops.db"
```

### If uvx Is Not Available in the Container

Use the npm-based SQLite MCP alternative:

```yaml
mcpServers:
  sqlite:
    title: "SQLite"
    description: "Query local databases — findings tracker, asset inventory, analytics"
    command: "npx"
    args:
      - "-y"
      - "@benborla29/mcp-server-sqlite"
      - "/app/data/mcp-sqlite/ops.db"
```

> **Note:** `@benborla29/mcp-server-sqlite` is a community npm port of the official Python server. Tool names and behavior are compatible. If this package is also unavailable or broken, the FastMCP wrapper approach below is the reliable fallback.

### FastMCP Wrapper (Reliable Fallback)

If neither uvx nor the npm alternative works, use a thin Python wrapper. This requires Python inside the LibreChat container or an external sidecar.

```python
# ~/LibreChat/data/mcp-tools/mcp_sqlite.py
from fastmcp import FastMCP
import sqlite3
from pathlib import Path

DB_PATH = Path("/app/data/mcp-sqlite/ops.db")
mcp = FastMCP("SQLite")

@mcp.tool()
def read_query(query: str) -> str:
    """Execute a SELECT query and return results."""
    conn = sqlite3.connect(DB_PATH)
    try:
        cursor = conn.execute(query)
        rows = cursor.fetchall()
        cols = [d[0] for d in cursor.description] if cursor.description else []
        return str({"columns": cols, "rows": rows})
    finally:
        conn.close()

@mcp.tool()
def write_query(query: str) -> str:
    """Execute INSERT/UPDATE/DELETE/CREATE statements."""
    conn = sqlite3.connect(DB_PATH)
    try:
        conn.execute(query)
        conn.commit()
        return "Query executed successfully."
    finally:
        conn.close()

@mcp.tool()
def list_tables() -> str:
    """List all tables in the database."""
    conn = sqlite3.connect(DB_PATH)
    try:
        rows = conn.execute("SELECT name FROM sqlite_master WHERE type='table'").fetchall()
        return str([r[0] for r in rows])
    finally:
        conn.close()

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

```yaml
# In librechat.yaml if using the wrapper:
mcpServers:
  sqlite:
    title: "SQLite"
    command: "python3"
    args: ["/app/data/mcp-tools/mcp_sqlite.py"]
```

### Available Tools (official server)

| Tool | Description |
|---|---|
| `list_tables` | Show all tables in the database |
| `describe_table` | Show schema (columns, types) |
| `read_query` | Execute SELECT queries |
| `write_query` | Execute INSERT/UPDATE/DELETE/CREATE |
| `create_table` | Create a new table |
| `append_insight` | Add an analytical observation to internal memo |

### Example Prompts

```
"Create a table called 'findings' with columns: id, region, cip_standard,
 description, severity, due_date, status, assigned_to"

"Insert a new finding: CIP-007-R2 gap on TX-ESX-03, high severity, due March 15"

"Show me all critical findings older than 30 days grouped by region"

"How many open findings do we have per CIP standard?"
```

### Data Location and Backup

```bash
# Database on Mac
ls -la ~/LibreChat/data/mcp-sqlite/ops.db

# Verify it's a real SQLite file (not a 0-byte placeholder)
sqlite3 ~/LibreChat/data/mcp-sqlite/ops.db "SELECT 1;" 2>/dev/null \
  && echo "✓ SQLite database valid" || echo "⚠ Re-initialize: sqlite3 ~/LibreChat/data/mcp-sqlite/ops.db 'SELECT 1;'"

# Backup
cp ~/LibreChat/data/mcp-sqlite/ops.db \
   ~/Backups/mcp-sqlite-$(date +%Y%m%d).db

# Migration from mcpo: copy from old location
# cp ~/data/ops.db ~/LibreChat/data/mcp-sqlite/ops.db
```

---

## Git MCP Server

**What:** Inspect git repositories — branches, commits, diffs, file contents, and code search from chat.

### Configuration in `librechat.yaml`

```yaml
mcpServers:
  git:
    title: "Git"
    description: "Inspect repos — branches, diffs, commit history, code review"
    command: "uvx"
    args:
      - "mcp-server-git"
      - "--repository"
      - "/host-home/Projects"
```

### If uvx Is Not Available

Use the npm alternative `@cyanheads/git-mcp-server`:

```yaml
mcpServers:
  git:
    title: "Git"
    command: "npx"
    args:
      - "-y"
      - "@cyanheads/git-mcp-server"
```

The `--repository` argument may differ — check the package documentation for the npm version.

### Available Tools

| Tool | Description |
|---|---|
| `git_status` | Show working tree status |
| `git_log` | View commit history |
| `git_diff` | Compare branches, commits, or working tree |
| `git_diff_staged` | Show staged changes |
| `git_diff_unstaged` | Show unstaged changes |
| `git_show` | Show contents of a specific commit |
| `git_list_branches` | List all local and remote branches |
| `git_list_files` | List all tracked files |
| `git_read_file` | Read a file at a specific branch/commit |
| `git_search_code` | Search for patterns across the codebase |

### Example Prompts

```
"Show me the git status of my /host-home/Projects/ot-automation repo"

"What changed in the last 5 commits to the bigfix-fixlets project?"

"Diff the main branch against the cip007-remediation branch"

"Search my codebase for any file that uses the Tenable API client"
```

**Note:** The git server accesses repos under the configured `--repository` path. If your repos are at `/host-home/Projects`, any repo under that directory is accessible.

---

## Sequential Thinking MCP Server

**What:** Externalizes the model's thought process. The model breaks problems into numbered steps, can revise previous steps, branch into alternatives, and adjust dynamically. The reasoning chain is visible and auditable.

### Configuration in `librechat.yaml`

```yaml
mcpServers:
  sequential-thinking:
    title: "Sequential Thinking"
    description: "Structured step-by-step reasoning with revision and branching"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-sequential-thinking"
```

### Chat Menu Option

To keep Sequential Thinking available only via Agent Builder (not cluttering the chat dropdown):

```yaml
mcpServers:
  sequential-thinking:
    title: "Sequential Thinking"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-sequential-thinking"
    chatMenu: false  # Only available in Agent Builder
```

### Available Tools

| Tool | Description |
|---|---|
| `sequentialthinking` | Submit a thought step with step number, total steps, and revision metadata |

### Example Prompts

```
"Think through the CIP-007 R2 compliance requirements step by step for our Linux
 fleet. What evidence do we need, what gaps might exist, and what's the remediation path?"

"Analyze this alert chain sequentially — what's the most likely attack path?"

"Debug this Python script step by step — it returns empty results from the Tenable API."

"Help me plan the migration from network scanning to agent-based scanning.
 Think through dependencies, risks, and rollout phases."
```

> **Model recommendation:** Use `auto-reasoning` (deepseek-r1:32b) or `auto` (qwen3:32b) for best Sequential Thinking results. Both models generate well-formed tool call schemas for this server.

> **Known issue:** Some model/LibreChat version combinations may produce HTTP 422 errors on the `revisesThought` field — a type-parsing issue. If this occurs, use qwen3:32b in standard mode and update LibreChat: `cd ~/LibreChat && git pull && docker compose up -d`.

---

## Time MCP Server

**What:** Accurate timezone conversions and current time queries. Delegates date/time math to a reliable backend rather than letting the LLM guess.

### Configuration in `librechat.yaml`

```yaml
mcpServers:
  time:
    title: "Time"
    description: "Timezone conversions and date math across TX/NY/CA regions"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-time"
      - "--local-timezone=America/Chicago"
```

> **Verify `--local-timezone` is a supported argument** before relying on it:
> ```bash
> # Test inside container:
> docker compose exec api npx -y @modelcontextprotocol/server-time --help 2>/dev/null | head -10
> ```
> If `--local-timezone` is not a supported CLI flag (the package may use env vars instead), replace with:
> ```yaml
> time:
>   command: "npx"
>   args: ["-y", "@modelcontextprotocol/server-time"]
>   env:
>     LOCAL_TIMEZONE: "America/Chicago"
> ```

### Available Tools

| Tool | Description |
|---|---|
| `get_current_time` | Get current time in any timezone |
| `convert_time` | Convert a time between timezones |

### Example Prompts

```
"What time is it right now in Texas, New York, and California?"

"If I schedule a meeting at 2pm Central, what time is that in Eastern and Pacific?"

"Convert 9am Tokyo time to Central time"
```

---

## Microsoft Learn (Remote — Zero Install)

**What:** Queries official Microsoft documentation. No local process, no install — just a URL in `librechat.yaml`.

### Configuration in `librechat.yaml`

```yaml
mcpServers:
  microsoft-learn:
    title: "Microsoft Learn"
    description: "Official Microsoft documentation grounding"
    type: streamable-http
    url: "https://learn.microsoft.com/api/mcp"
```

### Example Prompts

```
"How do I configure Windows Server 2025 to enforce SMB signing via Group Policy?"

"What's the current PowerShell cmdlet syntax for managing AD certificate templates?"

"Show me the official documentation for SQL Server Always On availability groups"
```

> **Security note:** Microsoft Learn queries go to Microsoft's servers. Don't send sensitive or classified data in prompts that include MS Learn context. The public endpoint covers most documentation without authentication.

---

## Memory Budget

| Component | Idle Memory | Active Memory | Notes |
|---|---|---|---|
| Filesystem server | ~40MB | ~45MB | Node.js process + npm runtime |
| Memory server | ~40MB | ~50MB | Node.js + JSON graph loaded |
| SQLite server | ~25MB | ~30MB | Python/Node + SQLite embedded |
| Git server | ~25MB | ~40MB | Python/Node + git ops |
| Sequential Thinking | ~40MB | ~40MB | Node.js process |
| Time server | ~35MB | ~35MB | Node.js process |
| **Total** | **~205MB** | **~240MB** | Similar to mcpo ecosystem |
| **vs mcpo stack** | **~235MB** | **~270MB** | Slight savings — no mcpo proxy overhead |

These processes run inside the LibreChat API container. Their memory is included in LibreChat's container memory budget, unlike the mcpo stack where they were separate host processes. The practical impact on your 64GB M4 is negligible either way.

---

## Managing MCP Servers

### Restart After Config Changes

```bash
cd ~/LibreChat

# After editing librechat.yaml — restart API container only (fast)
docker compose restart api

# After adding new volume mounts in docker-compose.override.yml — full restart
docker compose up -d
```

> **npm Cold-Start Performance:** The `docker-compose.override.yml` from the Setup Guide mounts a persistent `npm_cache` volume at `/root/.npm`. This means `npx -y @modelcontextprotocol/...` package checks are cached across container restarts. Without this volume, every LibreChat restart triggers npx to re-verify each MCP package against the npm registry (2-5 seconds per server, 6 servers = up to 30 seconds before tools appear). With the cache volume, packages are found locally and spawn instantly. If you set up the override file before this fix was documented, add the volume now:
>
> ```yaml
> # Add to services.api.volumes in docker-compose.override.yml:
>       - npm_cache:/root/.npm
>
> # Add at the bottom of docker-compose.override.yml:
> volumes:
>   npm_cache:
>     driver: local
> ```
> Then: `docker compose up -d` (volume creation requires a full restart, not just `restart api`)

### Check MCP Server Startup

```bash
# See which MCPs started successfully and which failed
docker compose logs api | grep -i mcp

# Detailed MCP debug output (if LibreChat has verbose logging enabled)
docker compose logs api | grep -i "mcp\|stdio\|filesystem\|memory\|sqlite\|git"
```

### Test Individual MCP Connections

```bash
# Test filesystem mount
docker compose exec api ls /host-home/Projects

# Test memory data directory
docker compose exec api ls /app/data/mcp-memory/

# Test SQLite file
docker compose exec api ls -la /app/data/mcp-sqlite/ops.db

# Test that npx is working (required for most MCP servers)
docker compose exec api npx --version
```

### Add a New MCP Server

1. Edit `~/LibreChat/librechat.yaml` — add entry under `mcpServers`
2. If stdio server needs a new directory mount, add to `docker-compose.override.yml`
3. Restart: `docker compose restart api` (or `docker compose up -d` if you added volumes)
4. Verify in LibreChat — the new server should appear in the chat tools dropdown

No new ports, no new Open WebUI connections, no mcpo config changes. LibreChat discovers new servers automatically from `librechat.yaml`.

---

## Adding Future Servers

LibreChat's native stdio support means adding any new stdio MCP server is as simple as adding it to `librechat.yaml`. No proxy infrastructure needed.

**Example — GitHub MCP server:**

```yaml
mcpServers:
  github:
    title: "GitHub"
    description: "GitHub repos, PRs, issues, code search"
    command: "npx"
    args:
      - "-y"
      - "@modelcontextprotocol/server-github"
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_your_token_here"
```

**Example — Notion MCP server:**

```yaml
mcpServers:
  notion:
    title: "Notion"
    description: "Read/write Notion pages and databases"
    command: "npx"
    args:
      - "-y"
      - "@notionhq/mcp"
    env:
      NOTION_API_KEY: "secret_your_key_here"
```

Both examples require restarting the LibreChat API container. Neither requires any infrastructure changes beyond the `librechat.yaml` edit.

---

## Troubleshooting

### MCP server fails to start (logs show spawn error)

```bash
# Check full logs
docker compose logs api | grep -i error | head -30

# Test spawning the command manually inside the container
docker compose exec api npx -y @modelcontextprotocol/server-filesystem /host-home/Projects
# Should wait for stdin — Ctrl+C to exit
# If it errors, the package or Node.js version has an issue
```

### SQLite or Git server fails (uvx not found)

```bash
# Check if uvx is available
docker compose exec api uvx --version

# If not found, switch to npm alternatives in librechat.yaml (see server sections above)
# Or install uvx inside the container (not persistent — needs Dockerfile customization):
# docker compose exec api pip install uv
```

### Memory not persisting between sessions

```bash
# Verify the file exists after a create_entities call
ls -la ~/LibreChat/data/mcp-memory/memory.json
# File is created on first write — not at server startup

# Verify the mount is correct
docker compose exec api ls /app/data/mcp-memory/

# If file exists on Mac but agent can't find it, check permissions:
chmod 644 ~/LibreChat/data/mcp-memory/memory.json
```

### Tools not appearing in chat dropdown

After restarting, wait ~30 seconds for LibreChat to spawn and connect to all MCP servers. Then refresh the browser. If tools still don't appear:

1. Check logs: `docker compose logs api | grep -i mcp | tail -20`
2. Verify `librechat.yaml` YAML syntax is valid: `python3 -c "import yaml; yaml.safe_load(open('librechat.yaml'))" && echo OK`
3. Ensure container can reach external packages: `docker compose exec api curl -s https://registry.npmjs.org/@modelcontextprotocol/server-memory/latest | python3 -m json.tool | head -3`

---

### Document Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0 | Feb 24, 2026 | Initial release — LibreChat native stdio MCP architecture. Covers all 6 core servers (Filesystem, Memory, SQLite, Git, Sequential Thinking, Time) plus Microsoft Learn remote. Documents uvx fallback options for SQLite/Git, FastMCP wrapper pattern, per-server troubleshooting, and migration paths from mcpo. |
| 1.1 | Feb 24, 2026 | **FIX:** SQLite initialization guidance updated — removed `touch` placeholder approach, added `sqlite3 ... "SELECT 1;"` verification command to Data Location section to confirm valid database header. **ADD:** npm cache volume context in Managing MCP Servers section — explains the `npm_cache` volume in `docker-compose.override.yml`, documents the 2-5s/server cold-start latency it prevents, and provides retrofit instructions for setups created before this fix. **FIX:** Companion doc header updated to reference LibreChat Stack Setup Guide v1.1. |

---

*LC_MCP_Core_Servers.md v1.1 — Companion to LibreChat Stack Setup Guide v1.1 · Phase 15*
