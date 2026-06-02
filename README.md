# agentmemory Setup Guide

Replicate our persistent memory system on any machine. Tested on Fedora 43, Node v24, Hermes Agent v0.15.1.

**Docs:**
- This file — installation and configuration
- [USAGE-GUIDE.md](USAGE-GUIDE.md) — how Hermes actually uses agentmemory in practice (operational patterns, decision tree, examples)

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

## Compression Providers

The compression engine calls an LLM to summarize observations. Any OpenAI-compatible API works.

### OpenAI (default)
```ini
Environment="OPENAI_API_KEY=sk-..."
Environment="OPENAI_BASE_URL=https://api.openai.com/v1"
```

### DeepSeek (tested, works with crystallize)
```ini
Environment="OPENAI_API_KEY=sk-..."
Environment="OPENAI_BASE_URL=https://api.deepseek.com/v1"
# Requires setting the model via ~/.agentmemory/.env (see below)
```

### Xiaomi Mimo
```ini
Environment="OPENAI_API_KEY=tp-sw1-..."
Environment="OPENAI_BASE_URL=https://token-plan-sgp.xiaomimimo.com/v1"
```

### DashScope (Alibaba/Qwen)
```ini
Environment="OPENAI_API_KEY=sk-..."
Environment="OPENAI_BASE_URL=https://coding-intl.dashscope.aliyuncs.com/v1"
```

## Daemon Environment (.env)

The daemon also reads `~/.agentmemory/.env` for runtime configuration. This file controls the compression model, features, and MCP tool visibility.

Create `~/.agentmemory/.env`:

```bash
# Compression model (must match OPENAI_BASE_URL provider)
OPENAI_MODEL=deepseek-v3-flash        # or gpt-4o-mini, qwen-turbo, etc.
OPENAI_MAX_TOKENS=8192

# Feature flags
GRAPH_EXTRACTION_ENABLED=true
CONSOLIDATION_ENABLED=true
AGENTMEMORY_AUTO_COMPRESS=true
AGENTMEMORY_INJECT_CONTEXT=true
SNAPSHOT_ENABLED=true

# MCP tool visibility
AGENTMEMORY_TOOLS=all                  # 'all' exposes all 53 tools; default is 8
```

**Note:** The systemd service Environment= variables take precedence over `.env`. If you set `OPENAI_API_KEY` in both places, the service file wins.

## Practical Usage Guide

These are the agentmemory MCP tools available in Hermes, organized by when to use them.

### Session Lifecycle

| When | Tool | Purpose |
|---|---|---|
| Start of complex task | `memory_action_create` | Create work item with dependencies |
| During work | `memory_action_update` | Track progress (pending → active → done) |
| End of action chain | `memory_crystallize` | Compress completed work into a digest |
| Periodically | `memory_consolidate` | Run 4-tier pipeline (working → episodic → semantic → procedural) |
| Periodically | `memory_reflect` | Synthesize higher-order insights from patterns |

### Knowledge Capture

| When | Tool | Purpose |
|---|---|---|
| Discover a pattern/lesson | `memory_lesson_save` | Save for future sessions (confidence-scored) |
| Important decision | `memory_save` | Persistent fact (architecture, bug, workflow) |
| Something fails repeatedly | `memory_lesson_save` | Lesson with high confidence |
| Edit files | `memory_file_history` | Track what changed in specific files |

### Recall (before acting)

| When | Tool | Purpose |
|---|---|---|
| Before deciding | `memory_recall` | Search past observations by keyword |
| Before deciding | `memory_smart_search` | Hybrid semantic + keyword search |
| Check prior lessons | `memory_lesson_recall` | Search lessons by query |
| Verify a memory | `memory_verify` | Trace citation chain to source |

### Multi-Agent Coordination

| When | Tool | Purpose |
|---|---|---|
| Message another agent | `memory_signal_send` | Typed message (info, request, alert, handoff) |
| Check messages | `memory_signal_read` | Read messages for your agent ID |
| Share knowledge | `memory_team_share` | Share memory/observation with team |
| See team activity | `memory_team_feed` | Recent shared items from all members |
| Sync across instances | `memory_mesh_sync` | Push/pull memories between agentmemory instances |

### Workflows & Gates

| When | Tool | Purpose |
|---|---|---|
| Complex dependencies | `memory_sentinel_create` | Event-driven watcher (webhook, timer, threshold) |
| External approval | `memory_checkpoint` | CI/deploy/approval gate |
| Trigger a gate | `memory_sentinel_trigger` | Externally fire a sentinel |
| Exploratory work | `memory_sketch_create` | Ephemeral action graph (auto-expires) |
| Commit exploratory work | `memory_sketch_promote` | Promote sketch to permanent actions |

### Inspection & Debug

| Tool | Purpose |
|---|---|
| `memory_diagnose` | Health checks across all subsystems |
| `memory_heal` | Auto-fix stuck actions, stale leases, orphaned data |
| `memory_audit` | Audit trail of memory operations |
| `memory_export` | Export all memory data as JSON |
| `memory_timeline` | Chronological observations around a date |
| `memory_profile` | User/project profile with top concepts |
| `memory_patterns` | Detect recurring patterns across sessions |
| `memory_insight_list` | List synthesized insights |
| `memory_graph_query` | Query the knowledge graph |

## Known Limitations (v0.9.24)

- **Slots bug:** `memory_slot_create` returns 500 in v0.9.24. Not a config issue — upstream bug.
- **Hooks are Claude Code only:** The agentmemory hooks (session-start, post-tool-use, etc.) are designed for Claude Code's hook system. In Hermes CLI, session observation capture works differently — Hermes has its own native memory system that persists between sessions. The MCP tools work fine regardless.
- **Compression model must match provider:** If `OPENAI_BASE_URL` points to DeepSeek, `OPENAI_MODEL` must be a DeepSeek model (e.g., `deepseek-v3-flash`). Mismatched model names cause 400 errors (non-fatal).
- **Crystallize/Reflect/Consolidate require daemon:** These tools call the daemon's LLM pipeline. The daemon must be running with correct `OPENAI_API_KEY` and `OPENAI_BASE_URL` in the service file or `.env`.

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
