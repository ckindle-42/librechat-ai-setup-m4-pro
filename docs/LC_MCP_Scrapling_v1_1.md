# MCP Server: Scrapling — Intelligent Web Fetching (LibreChat Edition)
### Companion document to LibreChat Stack Setup Guide v1.1 — Phase 15
**Document version:** 1.1 (Feb 24, 2026)

**Server:** Scrapling MCP  
**Port:** `:8900`  
**Transport:** Streamable HTTP (native)  
**Memory:** ~50MB idle (Fetcher tier) · ~500MB when browser tier activated  
**License:** BSD-3-Clause  
**Repository:** https://github.com/D4Vinci/Scrapling

---

## What This Does

Scrapling gives your local LLMs the ability to **fetch live web content from chat**. When you ask the model to help you write a script using the Tenable API, it can pull the current API reference docs, parse the relevant endpoints, and write code against live documentation — not stale training data.

It works through three escalation tiers, tried in order:

| Tier | Engine | Speed | Memory | Anti-Bot | Use Case |
|---|---|---|---|---|---|
| **Fetcher** | HTTP + TLS fingerprint impersonation | ~0.5s | ~50MB (base process) | Basic (User-Agent, TLS) | 80% of doc sites, APIs, GitHub, readthedocs |
| **DynamicFetcher** | Playwright / Chromium | ~3-5s | ~300MB | JS rendering, SPA support | JS-rendered SPAs, single-page doc sites |
| **StealthyFetcher** | Modified Firefox (Patchright) | ~5-10s | ~500MB | Cloudflare Turnstile, Akamai, DataDome | Protected vendor portals, gated docs |

The LLM starts with the fast path and only escalates when it gets blocked or empty content.

**Key advantage over raw web browsing:** Scrapling's MCP server lets the model pass CSS/XPath selectors to extract only the relevant content *before* it enters the context window. Instead of dumping a 50KB page into the prompt, the model can say "fetch this URL and extract only `.api-endpoint` elements" — dramatically reducing token waste.

---

## How LibreChat Connects to Scrapling

In the Open WebUI stack, Scrapling was added as a connection in the Admin Panel GUI. In LibreChat, it is declared directly in `librechat.yaml` as a native `streamable-http` MCP server:

```yaml
mcpServers:
  scrapling:
    title: "Scrapling Web Fetcher"
    description: "Fetch and parse web content with anti-bot bypass and CSS selector targeting"
    type: streamable-http
    url: "http://host.docker.internal:8900/mcp"
```

This entry is already present in the `librechat.yaml` created in Phase 4 of the LibreChat Stack Setup Guide. **No additional configuration in LibreChat is needed** — the MCP server appears automatically in the chat tools dropdown and Agent Builder when Scrapling is running.

---

## Prerequisites

- **LibreChat v0.7.x+** (MCP support, including streamable-http type)
- **Python 3.9+** (already in your stack)
- **Phase 5 router** running at `:8000`

Verify before proceeding:
```bash
python3 --version   # Must be 3.9+
pip --version       # Must be available
```

---

## Installation

### Step 1: Install Scrapling

```bash
# Install with all fetcher backends (Playwright + Patchright stealth)
pip install "scrapling[all]" --break-system-packages

# Install Playwright browsers
python -m playwright install chromium

# macOS note: playwright install-deps is a Linux-only command
# (installs apt packages). On macOS, Playwright's Chromium is self-contained.
# Do NOT run playwright install-deps on macOS — it will fail or no-op.

# Verify installation
scrapling --version
python -c "from playwright.sync_api import sync_playwright; print('Playwright OK')"
```

> ⚠️ **StealthyFetcher browser:** The stealth tier uses Patchright (patched Playwright)
> which shares the Chromium install. If you also want Camoufox (Firefox-based stealth),
> run `scrapling install camoufox` — but Patchright covers most cases and Camoufox has
> known memory issues on some macOS versions. Start without it.

### Step 2: Start the MCP Server

```bash
# Start in Streamable HTTP transport mode
# --host 127.0.0.1 binds to loopback only — not exposed to network
scrapling mcp --http --host 127.0.0.1 --port 8900

# Verify it's running
curl -s -o /dev/null -w "%{http_code}" http://localhost:8900/mcp
# Expected: 200
```

### Step 3: Create Launcher Script

```bash
cat > ~/launch_scrapling_mcp.sh << 'EOF'
#!/bin/zsh
# Scrapling MCP Server — Streamable HTTP on :8900
# Used by LibreChat (native streamable-http type in librechat.yaml)
# Also compatible with Open WebUI if ever needed

export PATH="$HOME/.local/bin:$PATH"

if lsof -i :8900 > /dev/null 2>&1; then
    echo "Scrapling MCP already running on :8900"
    exit 0
fi

nohup scrapling mcp --http --host 127.0.0.1 --port 8900 \
    > /tmp/scrapling-mcp.log 2>&1 &

echo $! > /tmp/scrapling-mcp.pid

for i in {1..10}; do
    if curl -s -o /dev/null -w "%{http_code}" http://localhost:8900/mcp 2>/dev/null | grep -q "200"; then
        echo "✓ Scrapling MCP → localhost:8900 (PID $(cat /tmp/scrapling-mcp.pid))"
        exit 0
    fi
    sleep 1
done

echo "✗ Scrapling MCP failed to start. Check /tmp/scrapling-mcp.log"
exit 1
EOF
chmod +x ~/launch_scrapling_mcp.sh
```

### Step 4: Launcher Integration

The `launch_ai_stack.sh` in Phase 12 of the LibreChat Stack Setup Guide already includes the Scrapling auto-start block:

```bash
# This is already in launch_ai_stack.sh — no manual addition needed:
if [ -f ~/launch_scrapling_mcp.sh ]; then
    if ! lsof -i :8900 > /dev/null 2>&1; then
        ~/launch_scrapling_mcp.sh
    fi
    ok "Scrapling MCP    → localhost:8900"
else
    warn "Scrapling MCP   → skipped (not installed — see LC_MCP_Scrapling.md)"
fi
```

No changes to `launch_ai_stack.sh` are needed.

---

## LibreChat Tool Access

### Chat Interface

When Scrapling is running and configured in `librechat.yaml`, its tools appear in the **Tools** dropdown below the chat input when using the M4 Stack or Ollama Direct endpoints. Click the tools icon → select "Scrapling Web Fetcher" → all six tools become available for that conversation.

### Agent Builder

In the LibreChat Agent Builder, select "Scrapling Web Fetcher" from the MCP Tools section when configuring agents. You can enable or disable individual Scrapling tools per agent (e.g., enable `get` and `bulk_get` but disable `stealthy_fetch` for a conservative research agent).

### Tool Names

| Tool | Description | Best For |
|---|---|---|
| `get` | Fast HTTP request with browser TLS fingerprint impersonation | Static pages, API docs, readthedocs, GitHub READMEs |
| `bulk_get` | Async parallel version of `get` for multiple URLs | Fetching 5-10 doc pages at once |
| `fetch` | Playwright/Chromium for dynamic content with full JS execution | JS-rendered SPAs, React/Vue doc sites |
| `bulk_fetch` | Async parallel browser fetch for multiple URLs | Multiple dynamic pages simultaneously |
| `stealthy_fetch` | Anti-bot bypass via stealth browser (Patchright) | Cloudflare-protected vendor portals |
| `bulk_stealthy_fetch` | Async parallel stealth fetch | Multiple protected pages |

All tools support CSS selector targeting, XPath selectors, timeout configuration, and proxy support.

---

## Usage Examples

### Example 1: Fetch API Documentation

```
You: I need to write a Python script that creates a scan in Tenable Security Center.
     Fetch the current API docs and help me build it.

Model: [calls Scrapling's get tool with Tenable SC API docs URL]
       [receives structured API reference]

       Based on the current Tenable SC API docs, here's the scan creation endpoint...
```

### Example 2: CSS Selector Extraction (Token-Efficient)

```
You: Fetch the python-pptx docs and get just the slide layout API reference.
     Use CSS selector "article" to extract only the main content.

Model: [calls get with URL and selector "article"]
       [receives ~2KB instead of ~50KB full page]

       The python-pptx slide layout API works like this...
```

> 💡 **CSS selectors reduce token waste dramatically.** Without them, the model
> receives the entire page HTML (often 50-100KB). With a targeted selector, it gets
> only the relevant fragment. Common selectors:
> - `article` or `main` — primary content (skips nav, footer)
> - `.api-endpoint` — API reference blocks
> - `#method-list` — specific section by ID
> - `table.parameters` — parameter tables

### Example 3: Agent with Scrapling

If you have a "Code Assistant" agent configured with Scrapling, it will automatically use web fetching when it determines current documentation is needed — without you having to ask:

```
You: [in Code Assistant agent] Write a BigFix Fixlet to detect CVE-2025-12345.
     I need current remediation guidance.

Agent: [autonomously calls Scrapling to fetch CVE details from NVD]
       [fetches BigFix reference docs for the relevant platform]
       [writes the Fixlet using live, current data]
```

---

## Memory Impact

| State | Memory | Notes |
|---|---|---|
| MCP server idle (Fetcher tier only) | ~50MB | Python process, no browser |
| Active `get`/`bulk_get` request | ~50MB | HTTP only, no browser launched |
| Active `fetch` request | ~300-500MB | Chromium spins up, exits after |
| Active `stealthy_fetch` request | ~400-800MB | Stealth browser, exits after |
| Browser cache between requests | ~100-200MB | Chromium stays warm ~60s |

On your 64GB M4, the Fetcher tier (80% of use cases) uses ~50MB. Browser tiers temporarily use 300-800MB that is released after the fetch completes.

---

## Troubleshooting

**Scrapling tools not appearing in LibreChat**

```bash
# 1. Verify Scrapling is running
curl -s -o /dev/null -w "%{http_code}" http://localhost:8900/mcp
# Must return 200

# 2. Restart LibreChat API to re-read librechat.yaml and reconnect MCPs
docker compose -f ~/LibreChat/docker-compose.yml restart api

# 3. Check LibreChat logs for MCP connection errors
docker compose -f ~/LibreChat/docker-compose.yml logs api | grep -i "scrapling\|mcp"

# 4. Verify host.docker.internal resolves from inside container
docker compose -f ~/LibreChat/docker-compose.yml exec api \
  curl http://host.docker.internal:8900/mcp -s -o /dev/null -w "%{http_code}"
# Must return 200
```

**MCP server won't start**

```bash
lsof -i :8900         # Check port in use
cat /tmp/scrapling-mcp.log   # Check startup logs
scrapling mcp --http --host 127.0.0.1 --port 8900  # Run in foreground to see errors
```

**`get` returns empty content (bot-blocked)**

The model should automatically escalate. If it doesn't, prompt: "That page returned empty, try using the browser-based fetch tool." Some sites require `stealthy_fetch` — tell the model explicitly if you know the site is protected.

**Chromium/Playwright not found**

```bash
python -m playwright install chromium
# If that fails:
pip install playwright --break-system-packages
python -m playwright install chromium
```

---

## Security Notes

- Scrapling binds to `127.0.0.1` only — not exposed to the network
- Web content fetched is passed to the LLM as context, same as copy-paste
- LibreChat accesses it via `host.docker.internal` — Docker container to host loopback
- Always respect `robots.txt` and website ToS — this tool is for fetching reference documentation, not mass scraping
- StealthyFetcher's anti-bot bypass is intended for accessing documentation behind overzealous bot protection, not for circumventing access controls

**SSRF Awareness:** Because the LLM controls what URLs Scrapling fetches, a prompt injection in untrusted content (e.g., a web page you ask the model to summarize) could attempt to direct Scrapling to fetch internal endpoints — for example, `http://localhost:8000/routing-info` or `http://host.docker.internal:27017`. Scrapling has no egress filtering. For a single-user local M4 deployment, this is an accepted risk. If you are ever running this stack in a multi-user or networked environment, add an egress allowlist at the reverse proxy layer (Caddy) or restrict Scrapling's network access via Docker networking.

---

## Comparison to Open WebUI Scrapling Setup

The Scrapling server itself is **identical** — same install, same launch script, same port, same tools. The only difference is how LibreChat discovers and connects to it:

| Aspect | Open WebUI | LibreChat |
|---|---|---|
| Connection setup | Admin Panel GUI → Add MCP Connection | Declared in `librechat.yaml` |
| URL used | `http://host.docker.internal:8900/mcp` | `http://host.docker.internal:8900/mcp` |
| Tool discovery | Auto on connection | Auto on container start |
| Tool assignment | Per-workspace checkboxes | Per-agent in Agent Builder |
| Connection type | MCP Streamable HTTP | `type: streamable-http` |

If you already have Scrapling installed and running from the Open WebUI stack, the same installation works for LibreChat without any changes.

---

### Document Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0 | Feb 24, 2026 | Initial release — LibreChat-specific Scrapling companion. Covers librechat.yaml integration, Agent Builder usage, chat dropdown access, and migration note for existing Open WebUI installations. Based on MCP_Scrapling_v1.3.md with all Open WebUI-specific sections replaced. |
| 1.1 | Feb 24, 2026 | **ADD:** SSRF awareness section in Security Notes — documents the LLM-controlled URL fetch risk for untrusted content scenarios and provides multi-user mitigation guidance. **FIX:** Companion doc header updated to reference LibreChat Stack Setup Guide v1.1. |

---

*LC_MCP_Scrapling.md v1.1 — Companion to LibreChat Stack Setup Guide v1.1 · Phase 15*
