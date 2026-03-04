# DELIA — Dynamic Enhanced Learning Integrated Assistant

> Modular AI agent framework — runs from the command line as a one-shot task, interactive REPL, or background daemon. Dynamically loads tools, skills, and connectors at runtime and routes requests to LLM providers via a local triage model. Features graph-based memory with hybrid vector+keyword search, multi-provider web search, multimodal vision, event-driven reactions, notification routing, parallel agent spawning, cron-style scheduling, a 4-slot plugin architecture, per-session tool tracking, and a mobile-responsive real-time web dashboard.

## Quick Start

```bash
# One-shot task
python -m agent run "create a Python hello-world project"

# Interactive REPL (Rich-powered terminal UI)
python -m agent chat

# Background daemon (HTTP API on port 8080)
python -m agent start --daemon --port 8080
python -m agent status
python -m agent stop

# Web UI (real-time dashboard with SSE streaming)
python -m agent ui --port 8081
python -m agent ui --host 0.0.0.0     # accessible on LAN

# Spawn a parallel agent in an isolated git worktree
python -m agent spawn "refactor auth module" --branch feat/auth

# Scheduled tasks (cron-style)
python -m agent cron add --cron "0 9 * * *" "check email and summarise"
python -m agent cron list
python -m agent cron pause <id>

# List / manage running agents
python -m agent agents list
python -m agent agents status <id>
python -m agent agents kill <id>

# Encrypt secrets at rest
python -m agent encrypt-secrets -c config.yaml
python -m agent decrypt-secrets -c config.yaml

# Diagnostics
python -m agent doctor
```

## Package Structure

```
DELIA/
├── config.yaml              # Project configuration
├── pyproject.toml           # Build config & dependencies
├── README.md                # ← you are here
│
└── agent/                   # Main package
    ├── __init__.py          # Package root (__version__)
    ├── __main__.py          # Entry: python -m agent
    ├── exceptions.py        # Typed exception hierarchy
    │
    ├── common/              # Shared utilities (DRY layer)
    │   ├── constants.py     # SKIP_DIRS, size limits, timeouts
    │   ├── file_utils.py    # Path safety, read/write, workspace walk
    │   ├── text_utils.py    # Truncation, encoding, line counting
    │   ├── notebook_utils.py# Cell finding, notebook I/O
    │   ├── mcp_utils.py     # MCP config, JSONRPC, transport
    │   ├── process_utils.py # Shell detection, subprocess wrappers
    │   └── types.py         # Shared TypedDicts, Enums, aliases
    │
    ├── config/              # Configuration system
    │   ├── schema.py        # Frozen dataclasses (ModelConfig, AgentConfig, …)
    │   ├── loader.py        # YAML parser, env-var expansion, secret decryption
    │   ├── encryption.py    # AES-GCM / Fernet encryption with auto key-gen
    │   └── secrets.py       # AES-GCM encrypted secret store
    │
    ├── core/                # Agent brain
    │   ├── agent.py         # Plan-act-observe loop with retry, routing, escalation
    │   ├── registry.py      # Dynamic tool discovery & dispatch
    │   ├── context.py       # Sliding-window conversation manager
    │   ├── memory.py        # Graph-based SQLite memory (nodes, edges, facts, FTS5, embeddings)
    │   ├── planner.py       # Task decomposition (PlanStep / Plan)
    │   ├── hooks.py         # Plugin hook system (9 lifecycle events)
    │   ├── compaction.py    # LLM-summarised context compression
    │   ├── session.py       # JSONL session persistence & resume
    │   ├── bootstrap.py     # User-editable .md prompt fragments
    │   ├── pruning.py       # Tool-result pruning (soft-trim / hard-clear)
    │   ├── retry.py         # Exponential backoff + jitter for LLM calls
    │   ├── ollama.py        # Ollama server auto-start & model pre-warming
    │   ├── reactions.py     # Event-driven reaction system (pattern → action)
    │   ├── notifier.py      # Multi-backend notification router
    │   ├── workspace.py     # Git worktree isolation & parallel agent spawning
    │   ├── sandbox.py       # Command sandboxing & risk classification
    │   ├── scheduler.py     # Cron-style task scheduling engine
    │   ├── self_improve.py  # Idle self-improvement cycles
    │   └── providers/       # LLM provider layer
    │       ├── base.py      # Message, Completion, ModelProvider Protocol
    │       ├── openai_compat.py # OpenAI-compatible HTTP client (stdlib only)
    │       └── router.py    # Multi-model triage, selection & escalation
    │
    ├── plugins/             # Plugin architecture (4-slot)
    │   ├── base.py          # PluginManifest, Plugin ABC, PluginRegistry
    │   └── github_tracker.py# Built-in: GitHub Issues tracker (REST API)
    │
    ├── tools/               # 60 tools (all in user_defined/)
    │   ├── base.py          # BaseTool ABC + ToolResult dataclass
    │   └── user_defined/    # Auto-discovered production tools
    │
    ├── connectors/          # External communication bridges
    │   └── base.py          # BaseConnector ABC + ConnectorRegistry
    │
    ├── skills/              # Dynamic skill packs
    │   ├── loader.py        # SkillLoader (discover, parse, activate)
    │   ├── agent-customization/ # Built-in: agent customisation primitives
    │   ├── gpu-stats/       # Built-in: GPU monitoring & stats
    │   └── web-research/    # Built-in: web research methodology
    │
    ├── cli/                 # Command-line interface (Rich-powered)
    │   ├── app.py           # argparse CLI with 15+ subcommands
    │   ├── repl.py          # Rich interactive REPL with /commands
    │   ├── daemon.py        # PID-based daemon + HTTP API
    │   └── doctor.py        # System diagnostics & health checks
    │
    └── ui/                  # Web dashboard (stdlib http.server + SSE)
        ├── server.py        # UIHandler — chat streaming, REST API, SSE events
        └── static/
            └── index.html   # Mobile-responsive dark-themed dashboard
```

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                      CLI / Daemon (Rich UI)                          │
│  run · chat · start · stop · status · config · tools · skills        │
│  ui · doctor · spawn · agents · cron · encrypt/decrypt-secrets       │
├──────────────────────────────────────────────────────────────────────┤
│                         Agent Core                                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐            │
│  │ Router   │ │ Planner  │ │ Memory   │ │ Context Mgr   │            │
│  │ (triage, │ │ (task    │ │ (graph + │ │ (sliding      │            │
│  │ escalate)│ │  decomp) │ │  hybrid) │ │  window)      │            │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────┘            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐            │
│  │ Hooks    │ │Compaction│ │ Sessions │ │ Bootstrap     │            │
│  │ (9 event │ │ (LLM     │ │ (JSONL   │ │ (.md prompt   │            │
│  │  points) │ │  summary)│ │  persist)│ │  fragments)   │            │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────┘            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐            │
│  │ Pruning  │ │ Retry    │ │Reactions │ │ Notifier      │            │
│  │ (tool    │ │ (backoff │ │ (event → │ │ (log/desktop/ │            │
│  │  results)│ │ + jitter)│ │  action) │ │  slack/email) │            │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────┘            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐            │
│  │Workspace │ │ Plugins  │ │Scheduler │ │ Sandbox       │            │
│  │ (git     │ │ (4-slot  │ │ (cron    │ │ (risk class,  │            │
│  │ worktree)│ │  system) │ │  engine) │ │  confinement) │            │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────┘            │
│  ┌──────────┐                                                        │
│  │Self-     │                                                        │
│  │Improve   │                                                        │
│  │ (idle)   │                                                        │
│  └──────────┘                                                        │
├──────────────────────────────────────────────────────────────────────┤
│                       Registry Layer                                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐          │
│  │ ToolRegistry │ │ SkillLoader  │ │ ConnectorRegistry    │          │
│  └──────────────┘ └──────────────┘ └──────────────────────┘          │
├──────────────────────────────────────────────────────────────────────┤
│               Provider Layer (OpenAI-compat)                         │
│  Clarifai · Ollama · vLLM · LiteLLM · any OpenAI-format API          │
├──────────────────────────────────────────────────────────────────────┤
│               Config / Secrets (YAML + AES-GCM encryption)           │
└──────────────────────────────────────────────────────────────────────┘
```

## Key Features

| Feature | Description |
|---------|-------------|
| **Local Triage Router** | Sub-second Ollama model (gemma3-1b) classifies task complexity (S/M/H) before routing to the appropriate cloud model; deterministic escalation without re-triaging on failures |
| **Graph Memory** | SQLite-backed knowledge graph with nodes, edges, facts, and FTS5 full-text search |
| **Hybrid Search** | Weighted merge of vector embeddings (cosine) + BM25 keyword search for memory recall |
| **Local Embeddings** | Ollama-powered `nomic-embed-text` model runs alongside the LLM — no external API needed |
| **Multimodal Vision** | Built-in vision tool analyses images via configured multimodal model (e.g. mm-poly-8b) |
| **Multi-Provider Web Search** | DuckDuckGo, Brave, Tavily, Jina, SearXNG with ordered fallback chains |
| **Cron Scheduling** | Cron-expression task scheduling with pause/resume and run history |
| **Event Reactions** | Pattern-matching reaction system — trigger actions on agent events automatically |
| **Notification Router** | Priority-based routing to log, desktop, webhook, Slack, or email backends with deduplication |
| **Parallel Agents** | Spawn isolated agents in git worktrees for concurrent work on different branches |
| **Plugin Architecture** | 4-slot system (Runtime, Tracker, Workspace, Notifier) with manifest-based discovery |
| **Web Dashboard** | Mobile-responsive, SSE-streamed chat with per-session tool tracking and message queuing |
| **Autonomy Controls** | Supervised/full modes, workspace confinement, command sandboxing, rate limiting |
| **Encrypted Secrets** | AES-GCM / Fernet encryption for secrets at rest in config files |
| **Self-Improvement** | Idle-time cycles that let the agent improve its own tools, skills, and memory |
| **60 Built-in Tools** | File I/O, search, git, browser, notebooks, MCP, GitHub, vision, and more — auto-discovered |

## Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Single Responsibility** | Each module does one thing — no god classes |
| **Dependency Inversion** | Core depends on `Protocol`/`ABC`, not implementations |
| **Open/Closed** | New tools/skills/connectors/plugins without modifying core |
| **DRY** | All shared constants and helpers live in `common/` |
| **Explicit Signatures** | Full type annotations and NumPy-style docstrings |
| **No Global State** | All state flows through injected config/context |
| **Resilient** | Retry with backoff, pruning, compaction, notifications, memory flush |

## Deployment

- **Docker** — `DEPLOY/docker/` includes Dockerfile, docker-compose.yml, and entrypoint script
- **Local install** — `DEPLOY/local/` includes install scripts for Windows (PowerShell) and Unix (bash)

## Requirements

- **Python 3.10+**
- **Dependencies:** `pyyaml>=6.0`, `rich>=13.0`, `playwright>=1.58`
- **Optional:** Ollama (local models + embeddings), git (worktree spawning)
- Cross-platform: Windows, macOS, Linux
