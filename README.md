# agentmemory Setup Guide

Replicate our persistent memory system on any machine. Tested on Fedora 43, Node v24, Hermes Agent v0.15.1.

## Architecture

```
┌─────────────┐     MCP (stdio)     ┌──────────────────┐     HTTP :3111     ┌──────────────────┐
│ Hermes Agent │ ◄─────────────────► │ @agentmemory/mcp │ ◄────────────────► │ agentmemory      │
│ (Python)     │                     │ (thin shim)      │                    │ daemon (iii-eng) │
└─────────────┘                      └──────────────────┘                    └────────┬─────────┘
                                                                                      │
                                                                              ┌───────┴───────┐
                                                                              │ ~/data/       │
                                                                              │ state_store.db│
                                                                              │ stream_store/ │
                                                                              └───────────────┘
```

**How it works:**
1. Hermes Agent spawns `@agentmemory/mcp` as a subprocess (MCP over stdio).
2. The MCP shim connects to the agentmemory daemon running on `http://localhost:3111`.
3. The daemon (iii-engine) manages persistent state in `~/data/`.
4. Hermes calls MCP tools (memory_save, memory_search, etc.) which proxy to the daemon.
5. The daemon also runs background tasks: graph extraction, consolidation, compression.

## Prerequisites

- Node.js v20+ (we use v24.14.0 via nvm)
- Hermes Agent installed
- An OpenAI-compatible API key (for LLM-based compression — can use any provider)

## Step 1: Install the daemon (systemd user service)

Create the service file at `~/.config/systemd/user/agentmemory.service`:

```ini
[Unit]
Description=agentmemory - Persistent memory server for AI coding agents
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=0

[Service]
Type=simple
ExecStart=<NODE_BIN_PATH>/npx -y @agentmemory/agentmemory
WorkingDirectory=<HOME>
Environment="PATH=<NODE_BIN_PATH>:<HOME>/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="HOME=<HOME>"
Environment="OPENAI_API_KEY=<YOUR_API_KEY>"
Environment="OPENAI_BASE_URL=<YOUR_BASE_URL>"
Environment="GRAPH_EXTRACTION_ENABLED=true"
Environment="CONSOLIDATION_ENABLED=true"
Environment="AGENTMEMORY_AUTO_COMPRESS=true"
Environment="AGENTMEMORY_INJECT_CONTEXT=true"
Restart=always
RestartSec=5
RestartMaxDelaySec=300
RestartSteps=5
KillMode=mixed
KillSignal=SIGTERM
TimeoutStopSec=30
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```

**Replace these placeholders:**

| Placeholder | Value | Example |
|---|---|---|
| `<NODE_BIN_PATH>` | Path to your node bin directory | `/home/user/.nvm/versions/node/v24.14.0/bin` |
| `<HOME>` | Your home directory | `/home/user` |
| `<YOUR_API_KEY>` | OpenAI-compatible API key | `sk-...` |
| `<YOUR_BASE_URL>` | API base URL (OpenAI or compatible) | `https://api.openai.com/v1` |

**To find your NODE_BIN_PATH:**
```bash
which npx
# /home/user/.nvm/versions/node/v24.14.0/bin/npx
# Use the directory part (without /npx)
```

**Note on OPENAI_BASE_URL:** The compression engine uses this to call an LLM for summarizing observations. Any OpenAI-compatible endpoint works (OpenAI, Xiaomi Mimo, DashScope, local vLLM, etc.). If the model isn't available at the URL, compression will fail (non-fatal — memory still works, just uncompressed).

## Step 2: Enable and start the service

```bash
# Reload systemd to pick up the new service
systemctl --user daemon-reload

# Enable auto-start on login
systemctl --user enable agentmemory.service

# Start now
systemctl --user start agentmemory.service

# Verify it's running
systemctl --user status agentmemory.service

# Check logs
journalctl --user -u agentmemory -f
```

The daemon will listen on `http://127.0.0.1:3111`. Data persists in `~/data/` (relative to WorkingDirectory in the service file).

## Step 3: Configure Hermes Agent

Edit `~/.hermes/config.yaml`:

### 3a. MCP Server Registration

Add agentmemory to `mcp_servers`:

```yaml
mcp_servers:
  agentmemory:
    command: npx
    args:
    - -y
    - '@agentmemory/mcp'
```

No `env` section needed — the MCP shim defaults to `http://localhost:3111` which matches the daemon port.

### 3b. Memory Provider

Set the memory section:

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  provider: agentmemory
  flush_min_turns: 6
  nudge_interval: 10
```

### 3c. Curator (optional but recommended)

The curator auto-manages skill lifecycle (staleness, archival):

```yaml
curator:
  enabled: true
  interval_hours: 168
  min_idle_hours: 2
  stale_after_days: 30
  archive_after_days: 90
  prune_builtins: true
  backup:
    enabled: true
    keep: 5
```

### 3d. Enable the MCP toolset

Make sure `agentmemory` is in your enabled toolsets or not in `disabled_toolsets`. In the `toolsets` section:

```yaml
toolsets:
- hermes-cli
- ask-user-question
```

And ensure it's NOT in `disabled_toolsets` under `agent:`.

## Step 4: Verify

```bash
# Restart Hermes to pick up config changes
hermes restart

# In a Hermes session, test:
# "save a test memory" → should call memory_save
# "search for test memory" → should call memory_search

# Check daemon health
curl -s http://localhost:3111/agentmemory/health | jq .
```

## Environment Variables Reference

| Variable | Required | Default | Description |
|---|---|---|---|
| `OPENAI_API_KEY` | Yes | — | API key for LLM-based compression |
| `OPENAI_BASE_URL` | No | `https://api.openai.com/v1` | API endpoint for compression LLM |
| `GRAPH_EXTRACTION_ENABLED` | No | `false` | Extract entity relationships from observations |
| `CONSOLIDATION_ENABLED` | No | `false` | Run memory consolidation pipeline |
| `AGENTMEMORY_AUTO_COMPRESS` | No | `false` | Auto-compress observations via LLM |
| `AGENTMEMORY_INJECT_CONTEXT` | No | `false` | Inject recalled context into agent prompts |
| `AGENTMEMORY_URL` | No | `http://localhost:3111` | Daemon URL (used by MCP shim) |

## Data Directory

The daemon stores data in `~/data/` (relative to systemd WorkingDirectory):

```
~/data/
├── state_store.db/     # SQLite-backed KV store (iii-engine StateModule)
│   ├── mem%3Amemories.bin
│   ├── mem%3Alessons.bin
│   ├── mem%3Asessions.bin
│   ├── mem%3Aobs%3A<session_id>.bin   # Per-session observations
│   ├── mem%3Aaudit.bin
│   ├── mem%3Aindex%3Abm25.bin         # BM25 search index
│   └── ...
└── stream_store/       # Event streams
```

**Backup:** Copy `~/data/` to preserve all memories, lessons, and sessions.

## Troubleshooting

### Compression errors in logs
```
Compression failed: Not supported model gpt-4o-mini
```
Your `OPENAI_BASE_URL` provider doesn't support `gpt-4o-mini`. Set `AGENTMEMORY_AUTO_COMPRESS=false` to disable, or point to a provider that supports the model.

### MCP tools not available in Hermes
Check that the MCP server is registered in `config.yaml` under `mcp_servers` and that the daemon is running on port 3111.

### Port conflict
If port 3111 is in use, change it in the iii-config (bundled inside the npm package) or set `AGENTMEMORY_URL` to a different port.

### Service won't start
```bash
journalctl --user -u agentmemory -n 50 --no-pager
```
Check for Node.js path issues or missing API key.

## Version Info (our current setup)

| Component | Version |
|---|---|
| @agentmemory/agentmemory | 0.9.24 |
| @agentmemory/mcp | 0.9.24 |
| Hermes Agent | 0.15.1 |
| Node.js | 24.14.0 |
| Python | 3.14.5 |
| OS | Fedora 43 (Linux 7.0.8) |

## Quick Install Script (one-liner)

For a fresh machine with Node.js and nvm already installed:

```bash
# Create systemd service (replace YOUR_API_KEY and YOUR_BASE_URL)
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/agentmemory.service << 'UNIT'
[Unit]
Description=agentmemory - Persistent memory server for AI coding agents
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=0

[Service]
Type=simple
ExecStart=%h/.nvm/versions/node/v24.14.0/bin/npx -y @agentmemory/agentmemory
WorkingDirectory=%h
Environment="PATH=%h/.nvm/versions/node/v24.14.0/bin:%h/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="HOME=%h"
Environment="OPENAI_API_KEY=YOUR_API_KEY"
Environment="OPENAI_BASE_URL=YOUR_BASE_URL"
Environment="GRAPH_EXTRACTION_ENABLED=true"
Environment="CONSOLIDATION_ENABLED=true"
Environment="AGENTMEMORY_AUTO_COMPRESS=true"
Environment="AGENTMEMORY_INJECT_CONTEXT=true"
Restart=always
RestartSec=5
RestartMaxDelaySec=300
RestartSteps=5
KillMode=mixed
KillSignal=SIGTERM
TimeoutStopSec=30
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
UNIT

systemctl --user daemon-reload
systemctl --user enable --now agentmemory.service
echo "agentmemory installed. Check: systemctl --user status agentmemory"
```
