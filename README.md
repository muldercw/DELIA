# DELIA — Dynamic Enhanced Learning Integrated Assistant

> Modular AI agent framework with a unified assistant runtime. DELIA runs as a one-shot task, interactive REPL, web dashboard, or background daemon. It dynamically loads tools, skills, and connectors at runtime, routes work across models, and can keep long-running mission profiles moving forward over time. Features graph-based memory with hybrid vector+keyword search, multi-provider web search, multimodal vision, event-driven reactions, notification routing, parallel agent spawning, cron scheduling, autonomous mission continuation, a 4-slot plugin architecture, per-session tool tracking, and a mobile-responsive real-time web dashboard.

## Installation

```bash
# Core runtime
pip install -e .

# Development tools
pip install -e ".[dev]"

# Optional voice connector stack
pip install -e ".[voice]"

# Recommended voice extras for the default backend choices
pip install faster-whisper resemblyzer

# Create your working config
copy "config.example.yaml" config.yaml
```

## Quick Start

```bash
# Unified assistant runtime
python -m agent

# One-shot task
python -m agent run "create a Python hello-world project"

# Interactive REPL only (without assistant automation extras)
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
python -m agent cron add "0 9 * * *" "check email and summarise"
python -m agent cron list
python -m agent cron pause <id>

# Mission profiles (persistent autonomous work)
python -m agent missions list
python -m agent missions show "Daily Check"
python -m agent missions run "Daily Check"
python -m agent missions history "Daily Check"

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
    │   ├── missions.py      # Mission profiles, ledger, continuation engine, lanes
    │   ├── self_improve.py  # Idle self-improvement cycles
    │   └── providers/       # LLM provider layer
    │       ├── base.py      # Message, Completion, ModelProvider Protocol
    │       ├── openai_compat.py # OpenAI-compatible HTTP client
    │       └── router.py    # Multi-model triage, selection & escalation
    │
    ├── plugins/             # Plugin architecture (4-slot)
    │   ├── base.py          # PluginManifest, Plugin ABC, PluginRegistry
    │   └── github_tracker.py# Built-in: GitHub Issues tracker (REST API)
    │
    ├── tools/               # Auto-discovered tool collection
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
│                      CLI / Daemon                                    │
│  run · chat · start · stop · status · config · tools · skills        │
│  ui · doctor · spawn · agents · cron · missions · encrypt/decrypt    │
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
│  ┌──────────┐ ┌──────────────┐                                       │
│  │Self-     │ │ Missions     │                                       │
│  │Improve   │ │ (profiles,   │                                       │
│  │ (idle)   │ │  ledger,     │                                       │
│  └──────────┘ │  continue)   │                                       │
│               └──────────────┘                                       │
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

## How DELIA Works

At a high level, DELIA is one runtime with several entry points:

- `agent` or `python -m agent` starts the full unified assistant runtime.
- `agent run` executes one task and exits.
- `agent chat` starts the interactive terminal REPL.
- `agent ui` starts the browser dashboard.
- `agent start --daemon` runs DELIA as a background service with HTTP endpoints plus recurring automation.

All of those entry points feed into the same core runtime in [agent/core/agent.py](agent/core/agent.py). That means chat, one-shot runs, the web UI, missions, and delegated workers all share the same basic reasoning loop, tool system, memory system, and provider routing.

The runtime flow is:

1. **Load config** from [config.yaml](config.yaml), decrypt secrets if needed, and build the immutable `AgentConfig` object.
2. **Discover capabilities** by loading tools, skills, connectors, and plugins from the workspace package tree.
3. **Choose a model** through the routing layer. A lightweight triage model can classify work first, then DELIA picks the best enabled model for the task.
4. **Run the agent loop** in `plan → act → observe → update context` cycles until the task is complete, the iteration budget is reached, or a timeout fires.
5. **Persist useful state** into sessions, memory, scheduler state, mission history, and notification records under `.agent/`.

### What the core loop is actually doing

Inside a normal run, DELIA repeatedly does this:

1. reads the current request plus recent conversation state
2. checks long-term memory for relevant prior facts
3. inspects the active tool schemas already available in the current turn
4. activates more tools from the tool library if a needed capability is not active yet
5. picks the right model/provider through the router
6. calls one tool at a time, observes the result, and decides the next step
7. prunes oversized tool output before feeding it back to the model
8. compacts old context only when the conversation is large enough to justify it
9. writes back useful outcomes to memory, session transcripts, and runtime state

### Unified orchestration

DELIA now treats long-running automation as first-class runtime capabilities instead of keeping them limited to CLI entry points.

That means the main agent can directly use:

- `mission_control` — inspect, run, queue, pause, resume, and review mission profiles
- `agent_fleet` — spawn, inspect, kill, and clean up delegated agents in isolated workspaces

In practice, this lets DELIA choose between:

- solving the task in the current turn
- delegating recurring work to a named mission
- spawning a parallel worker agent for isolated or long-running work

### How DELIA decides between local work, missions, and delegated agents

The runtime now uses a simple orchestration model:

- **Stay in the current turn** when the task is short, direct, and can be completed in one interactive pass.
- **Use `mission_control`** when the work should continue over time, matches a named mission profile, needs pause/resume behavior, or belongs in recurring background automation.
- **Use `agent_fleet`** when the work is better isolated in a separate workspace, should run in parallel, may touch a separate branch/worktree, or is long-running enough that it should not block the main conversation.

In other words:

- missions are for **persistent/resumable automation**
- delegated agents are for **parallel isolated execution**
- normal chat/tool use is for **immediate interactive completion**

When DELIA delegates work, it can later inspect state again through the same orchestration tools rather than treating the delegated work as a disconnected subsystem.

### Core runtime pieces

- **CLI / UI layer** — accepts input, starts sessions, and shows output.
- **Agent core** — the main reasoning loop in [agent/core/agent.py](agent/core/agent.py).
- **Registry layer** — discovers tools, skills, and connectors without hardcoding them.
- **Provider layer** — talks to OpenAI-compatible model endpoints.
- **Persistence layer** — keeps memory, sessions, cron tasks, and mission state on disk.

### What happens during a normal task

When you run a task in chat, UI, or one-shot mode, DELIA typically:

1. Reads bootstrap context and recent session context.
2. Recalls matching memory entries.
3. Builds a plan when the task is large enough.
4. Calls tools as needed.
5. Prunes oversized tool outputs before resending them to the model.
6. Compacts context if the conversation is getting too large.
7. Saves the final result and any useful memory facts.

### Where DELIA stores state

Most persistent runtime state lives under `.agent/` inside the workspace. The important parts are:

- `.agent/sessions/` — JSONL conversation transcripts and delegated-agent session state
- `.agent/memory/` or the configured memory database location — long-term graph memory
- `.agent/missions/` — mission state, history, and optional isolated mission workspaces
- `.agent/cron/` or scheduler state files — recurring scheduled task state
- `.agent/todos/` or session-scoped task state — large implementation task tracking when used

This layout is what makes DELIA resumable: the same workspace can keep memory, missions, delegated runs, and operator sessions connected over time.

## Autonomous Missions

DELIA now supports persistent mission profiles for work that should continue over time instead of relying on a single chat session.

Mission support lives in [agent/core/missions.py](agent/core/missions.py) and is built around four parts:

- `MissionProfile` — the mission definition loaded from `MISSION.yaml`
- `MissionLedger` — persisted state and run history under `.agent/missions/`
- `LaneController` — lightweight concurrency limits for background work
- `ContinuationEngine` — queues, schedules, triggers, and executes mission runs

### Mission lifecycle

1. DELIA scans configured `mission_dirs` for `MISSION.yaml` files.
2. Each file becomes a `MissionProfile`.
3. The engine restores prior state from `.agent/missions/state.json`.
4. A mission can be triggered manually, by schedule, continuously, or by incoming events.
5. The engine builds a mission brief using the objective, prior outcome, last blocker, optional `PLAYBOOK.md`, and optional `CHECKLIST.md`.
6. DELIA runs that brief through the normal agent runtime.
7. The result is written to the ledger, history is appended, and the next wake time is calculated.

### Mission files

Mission profiles are discovered from the configured mission directories. The default location is `.agent/missions/`.

Example structure:

```text
.agent/
  missions/
    daily-check/
      MISSION.yaml
      PLAYBOOK.md
      CHECKLIST.md
```

Example `MISSION.yaml`:

```yaml
name: Daily Check
description: Keep the workspace healthy.
objective: Review the current repo state, identify the most important issue, and move it forward.
mode: scheduled
schedule: every 30m
workspace_strategy: shared
cooldown_seconds: 300
max_consecutive_failures: 3
notify_on: [success, completed]
```

Supported mission modes:

- `manual` — only runs when explicitly started
- `scheduled` — runs from `schedule`
- `continuous` — keeps waking up on a repeating interval
- `event_driven` — reacts to matching events
- `blended` — supports both schedule-based and event-based triggers

### Mission control inside the agent

Mission orchestration is no longer only a CLI feature. The main agent can call the `mission_control` tool directly to:

- list available missions
- inspect mission state
- run a mission immediately
- queue a mission for background continuation
- pause or resume a mission
- review mission history

This makes mission execution part of the normal tool orchestration model.

Practically, this means a normal chat run can decide that a request belongs to an existing mission profile, queue it, and then come back later to inspect the mission's status or history.

### Mission commands

| Command | Description |
|---------|-------------|
| `agent missions list` | List discovered mission profiles |
| `agent missions show <mission>` | Show full profile + current state |
| `agent missions run <mission>` | Run a mission immediately |
| `agent missions pause <mission>` | Pause a mission |
| `agent missions resume <mission>` | Resume a mission |
| `agent missions history <mission>` | Show recent mission runs |

### Mission persistence

Mission state is stored under `.agent/missions/`:

- `.agent/missions/state.json` — current per-mission state
- `.agent/missions/history/<mission>.jsonl` — append-only run history
- `.agent/missions/workspaces/` — optional isolated mission workspaces

### Mission execution in daemon mode

When `agent start --daemon` runs and missions are enabled in config:

- the cron scheduler starts if cron tasks exist
- the mission continuation engine starts if mission profiles exist
- manual `/run` HTTP requests still work normally
- mission and cron automation run alongside the daemon process

## Delegated Agents

DELIA also supports first-class delegated agents through the `agent_fleet` tool and the existing spawn/session infrastructure.

Delegated agents use isolated workspaces managed by [agent/core/workspace.py](agent/core/workspace.py) and can be used when the main agent wants to offload work that is:

- parallelizable
- long-running
- safer in an isolated worktree
- better handled as a separate execution lane

The important distinction from missions is that delegated agents are usually about **execution isolation**, while missions are usually about **continuity over time**.

### Delegated agent lifecycle

1. The main agent or operator creates a delegated task.
2. DELIA creates a session record under `.agent/sessions/`.
3. A new isolated workspace is prepared.
4. The delegated agent runs in the background.
5. DELIA can later inspect status, read result state, kill, or clean up the session.

This is the same underlying idea exposed by both:

- CLI commands like `agent spawn` / `agent agents ...`
- the web UI and daemon surfaces
- the always-active `agent_fleet` runtime tool available to the main agent itself

### First-class orchestration tools

These tools are always active in the runtime:

| Tool | Purpose |
|------|---------|
| `mission_control` | Mission list/status/run/queue/pause/resume/history |
| `agent_fleet` | Spawn/list/status/kill/cleanup delegated agents |

This gives DELIA one unified orchestration model across chat, daemon, missions, and delegated workers.

## Web Dashboard

The built-in web UI (`python -m agent ui`) provides a real-time interface:

- **Chat** — SSE-streamed responses with live model indicators, thinking display, structured per-model token usage, and inline file downloads
- **Message queuing** — messages sent while the agent is responding are staged in chat and automatically dispatched when the current response completes
- **Left panel** — Skills and Tools with per-session highlighting of used tools
- **Right panel** — Models (with triage/multimodal badges) and Sessions (switch, create, delete)
- **Footer drawer** — Status cards, task progress, memory facts, per-model token breakdowns, and a unified orchestration pane
- **Orchestration pane** — inspect mission state, run/queue/pause/resume missions, see delegated-agent status, spawn new worker agents, and kill/clean up existing workers from one place
- **Mobile responsive** — hamburger menu reveals sidebar sections in a slide-in panel; viewport is locked to prevent scroll overflow

Access from other devices on your LAN with `--host 0.0.0.0`.

## Voice Connector

DELIA includes a local voice connector for always-on, wake-word-driven interaction.

What it supports:

- wake-word activation
- speech-to-text
- text-to-speech replies
- ambient conversation memory
- optional speaker verification / voice-print matching
- spoken restatement before TTS for more natural replies

It can also learn the user's voice over time. When `speaker_verify` is enabled, DELIA can enroll voice samples, build a voice-print, and use that profile to prefer or require the recognized speaker before acting on wake-word requests.

### Install

Start with the voice extra:

```bash
pip install -e ".[voice]"
```

For the default recommended setup, also install:

```bash
pip install faster-whisper resemblyzer
```

Recommended defaults in this repo:

- STT: `faster-whisper`
- TTS: `edge` or `piper`
- VAD: `silero`
- speaker verification: `resemblyzer`

### How it starts

If [config.yaml](config.yaml) contains a `connectors.voice` section and `assistant.auto_enable_voice: true`, the unified assistant runtime will auto-start voice when you launch:

```bash
python -m agent
```

To disable it for one run:

```bash
python -m agent --no-voice
```

To start voice explicitly in REPL mode:

```bash
python -m agent chat --connector voice
```

### Minimal config

```yaml
assistant:
  enabled: true
  auto_enable_voice: true

connectors:
  voice:
    wake_word: "computer"
    stt_backend: "faster-whisper"
    stt_model: "tiny"
    tts_backend: "edge"
    tts_voice: "en-US-AriaNeural"
    vad_backend: "silero"
    ambient_enabled: true
    restate_enabled: true
```

### Useful voice settings

- `wake_word` — phrase that activates DELIA
- `stt_backend` — `faster-whisper`, `moonshine`, or `vosk`
- `tts_backend` — `edge` or `piper`
- `vad_backend` — `silero` or `webrtc`
- `ambient_enabled` — store overheard room context
- `speaker_verify` — require a matching enrolled voice
- `speaker_enroll_samples` — number of samples used to learn the user's voice-print
- `speaker_threshold` — similarity threshold for accepting a speaker match
- `restate_enabled` — summarize long agent output into spoken form

See [agent/connectors/voice.py](agent/connectors/voice.py) and the `connectors.voice` section in [config.example.yaml](config.example.yaml) for the full set of voice options.

## CLI Commands

| Command | Description |
|---------|-------------|
| `agent` | Start the unified assistant runtime |
| `agent run "task"` | Execute a task and exit |
| `agent chat` | Start interactive REPL (Rich UI) |
| `agent start -d` | Start background daemon |
| `agent stop` | Stop the daemon |
| `agent status` | Show daemon status |
| `agent config` | Validate and display config |
| `agent tools [-f family]` | List registered tools |
| `agent skills` | List discovered skills |
| `agent ui [--host H] [-p PORT]` | Start the web dashboard |
| `agent doctor` | Run diagnostics (config, providers, tools, memory) |
| `agent spawn "task"` | Spawn a parallel agent in a git worktree |
| `agent agents list\|status\|kill\|cleanup` | Manage running agent fleet |
| `agent cron add\|list\|remove\|pause\|resume\|history` | Scheduled tasks |
| `agent missions list\|show\|run\|pause\|resume\|history` | Mission profiles and continuation |
| `agent encrypt-secrets` | Encrypt plaintext secrets in config.yaml |
| `agent decrypt-secrets` | Decrypt secrets back to plaintext |

Global flags: `-c CONFIG`, `-w WORKSPACE`, `-v` (info), `-vv` (debug), `-V` (version), `--profile`, `--no-voice`, `--no-ui`, `--ui-host`, `--ui-port`.

### REPL Commands

| Command | Description |
|---------|-------------|
| `/tools` | List all registered tools |
| `/memory` | Show persistent memory entries |
| `/sessions` | List saved sessions |
| `/resume <id>` | Resume a previous session |
| `/compact` | Force context compaction |
| `/status` | Show session, tokens, pruner, retry info |
| `/assistant` | Show unified assistant runtime status |
| `/activity` | Show recent assistant runtime activity |
| `/approvals` | Show approval queue state |
| `/approve <id> [note]` | Approve a pending action |
| `/reject <id> [note]` | Reject a pending action |
| `/context` | Token budget breakdown with visual bar |
| `/bootstrap` | Show loaded bootstrap files |
| `/hooks` | Show registered lifecycle hooks |
| `/improve` | Force a self-improvement cycle now |
| `/clear` | Clear context and start new session |
| `/help` | Show all commands |

## Configuration

The agent reads `config.yaml` from the project root.

Start by copying the example config to `config.yaml`, then fill in your model and secret values.

If you are using the repository as-is, copy [config.example.yaml](config.example.yaml) to [config.yaml](config.yaml).

```yaml
secrets:
  clarifai:
    CLARIFAI_PAT: "your-pat-here"

models:
  llm:
    gemma3-1b-local:              # Local Ollama model for instant triage/routing
      api_key: "ollama"
      base_url: "http://localhost:11434/v1"
      model: "gemma3:1b"
      provider: "ollama"
      temperature: 0
      supports_tool_use: false
    gpt-5-nano:                   # Lightweight cloud model (tool-capable)
      api_key: "your-pat-here"
      base_url: "https://api.clarifai.com/v2/ext/openai/v1"
      model: "gpt-5-nano"
      provider: "clarifai"
      temperature: 0.7
      supports_tool_use: true
    gpt-oss-20b:                  # Mid-tier cloud reasoning model
      api_key: "your-pat-here"
      base_url: "https://api.clarifai.com/v2/ext/openai/v1"
      model: "gpt-oss-20b"
      provider: "clarifai"
      temperature: 0.7
      supports_tool_use: true
  multimodal:
    model: "mm-poly-8b"
    provider: "clarifai"

agent:
  triage_model: "gemma3-1b-local"
  max_iterations: 200
  timeout_seconds: 180
  compaction_enabled: true
  session_persistence: true
  bootstrap_enabled: true
  pruning_soft_max_chars: 8000
  pruning_hard_max_chars: 50000
  retry_max_attempts: 3

assistant:
  enabled: true
  profile: hybrid
  auto_enable_voice: true
  proactive_review_enabled: true
  proactive_review_interval_minutes: 20

memory:
  embedding_provider: "openai"
  embedding_model: "nomic-embed-text"
  embedding_api_key: "ollama"
  embedding_base_url: "http://localhost:11434/v1"
  vector_weight: 0.7

search:
  provider: "duckduckgo"

autonomy:
  level: "supervised"
  workspace_only: true
  block_high_risk_commands: true

notifications:
  enabled: true
  backends: [log]

browser:
  enabled: true
  mode: "isolated"

self_improve:
  enabled: true
  interval_minutes: 60

missions:
  enabled: true
  mission_dirs:
    - ".agent/missions"
  tick_interval_seconds: 30
  max_concurrent_runs: 1
  default_cooldown_seconds: 300
  history_limit: 100
```

Secrets are injected as environment variables at load time and masked in logs. Values can be AES-GCM encrypted at rest — run `agent encrypt-secrets` to encrypt all plaintext secrets in your config file.

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
| **Mission Continuation** | Persistent mission profiles with schedule/event/manual triggering, run history, cooldowns, and background continuation |
| **Event Reactions** | Pattern-matching reaction system — trigger actions on agent events automatically |
| **Notification Router** | Priority-based routing to log, desktop, webhook, Slack, or email backends with deduplication |
| **Parallel Agents** | Spawn isolated agents in git worktrees for concurrent work on different branches |
| **Plugin Architecture** | 4-slot system (Runtime, Tracker, Workspace, Notifier) with manifest-based discovery |
| **Web Dashboard** | Mobile-responsive, SSE-streamed chat with per-session tool tracking and message queuing |
| **Autonomy Controls** | Supervised/full modes, workspace confinement, command sandboxing, rate limiting |
| **Encrypted Secrets** | AES-GCM / Fernet encryption for secrets at rest in config files |
| **Self-Improvement** | Idle-time cycles that let the agent improve its own tools, skills, and memory |
| **Auto-Discovered Tools** | File I/O, search, git, browser, notebooks, MCP, GitHub, vision, and more |

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

See the [`DEPLOY/`](DEPLOY/) directory for deployment options:

- **Docker** — `DEPLOY/docker/` includes Dockerfile, docker-compose.yml, and entrypoint script
- **Local install** — `DEPLOY/local/` includes install scripts for Windows (PowerShell) and Unix (bash)

## Requirements

- **Python 3.10+**
- **Dependencies:** `pyyaml>=6.0`, `rich>=13.0`, `playwright>=1.58`
- **Optional:** Ollama (local models + embeddings), git (worktree spawning)
- Cross-platform: Windows, macOS, Linux
