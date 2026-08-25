<div align="center">

# CodeMate

**A lightweight local coding agent built for real codebases**

Long-context management · Sub-agent collaboration · Persistent memory · Permission & reliability guardrails

**English** · [简体中文](./README.zh-CN.md)

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![Python](https://img.shields.io/badge/python-3.10%2B-3776AB.svg)](https://www.python.org/)
[![Polyglot Pass@1](https://img.shields.io/badge/Polyglot%20Pass%401-68.0%25-brightgreen.svg)](#evaluation)

<img src="docs/assets/demo.gif" alt="CodeMate TUI demo" width="760">

</div>

---

## Introduction

CodeMate is a coding agent that runs in your local terminal. It isn't built around "the model can call tools" — models already do that. It's built around **harness engineering**: the things that actually derail an agent on long-horizon tasks. Context blowing up. Subtasks polluting the main thread. Experience that can't be reused across sessions. Tool calls stepping outside their boundary. Loops that nobody stops.

Put simply: **the model does the thinking, the harness makes sure it finishes, stays stable, and stays safe.**

If you're building your own agent, CodeMate also works as a readable reference implementation — every mechanism has a concrete home in the source, rather than being a promise written into a prompt.

### Core capabilities

| Capability | What it does |
| --- | --- |
| **Layered context management** | L0–L2 compaction driven by message-count and token ceilings: offload large results to disk with a restore path → trim and archive history → model-generated summaries. Balances compaction cost against information loss, with a `prompt_too_long` recovery strategy as the backstop |
| **Sub-agent collaboration** | A `delegate` mechanism runs subtasks in isolated contexts and returns only the final conclusion. Four roles — researcher / reviewer / tester / coder — each get their own prompt and tool permissions (read-only / writable / executable) for least-privilege isolation |
| **Background execution** | Sub-agents and long-running commands can be dispatched to background threads; results are injected back on completion. The main loop never blocks, and independent tasks run in parallel |
| **Persistent memory** | Memory is persisted across three files — user / feedback / project — with write admission checked against a persistent / current_task dual scope. Extraction runs asynchronously after each turn, followed by similarity dedup, conflict merging, and snapshot-based safe writes |
| **Session persistence** | Full session traces are recorded as JSONL, enabling `resume` after an interruption and `share` for reproducing a run |
| **Permission system** | Deny / Ask / Allow policies, protected-path enforcement, agent-mode permission filtering, and an interactive authorization flow — keeping model-generated tool calls inside a verifiable, authorized execution boundary |
| **Reliability guardrails** | read-before-edit invariant, doom-loop detection, per-tool failure budget (writes `attempts_left / max_attempts` back into the error message so the model can self-correct), request budget (converts to an Interrupt at the ceiling and asks whether to continue), and an output clipper |
| **TUI** | Terminal interface with streaming output, collapsible tool calls, a background-task panel, permission prompts, and live context/cost meters |

### Evaluation

A reproducible evaluation pipeline built on the [Polyglot benchmark](https://github.com/Aider-AI/polyglot-benchmark) (C++ / Go / Java / JavaScript / Python / Rust). A stratified sample of 30 tasks serves as the dev set for iteration; a separate 100-task held-out set is run only at final release to avoid overfitting the benchmark. JSONL traces are the data source for a failure-mode taxonomy — context loss / malformed edits / infinite loops / tool misuse / budget exhaustion — which drives each harness iteration.

| Metric | Result |
| --- | --- |
| Average injected context | **↓ 81%** |
| Hard-failure rate from context exhaustion | **32.0% → 3.0%** |
| End-to-end task latency with parallel execution | **↓ 35%** |
| Relative Pass@1 gain from the guardrail system | **+21%** |
| Polyglot held-out (100 tasks) Pass@1 | **68.0%** |

> See [`eval/README.md`](./eval/README.md) for reproduction steps.

---

## Quick start

### Requirements

- Python **3.10+**
- Git
- An LLM API key (OpenAI / Anthropic / DeepSeek / any OpenAI-compatible endpoint)
- Language toolchains for whatever you want the test tools to run (`go`, `cargo`, `javac`, `pytest`, …)

### Install

```bash
# From source (recommended)
git clone https://github.com/<your-name>/CodeMate.git
cd CodeMate
pip install -e .

# Or with uv
uv pip install -e .
```

### Configure a model

The fastest path is environment variables:

```bash
export CODEMATE_API_KEY="sk-..."
export CODEMATE_BASE_URL="https://api.deepseek.com/v1"   # optional, OpenAI-compatible endpoint
export CODEMATE_MODEL="deepseek-chat"
```

Or run the interactive setup, which writes `~/.codemate/config.yaml`:

```bash
codemate init
```

### Run

```bash
cd /path/to/your/repo

# Launch the TUI (recommended)
codemate

# One-shot task, exits when done
codemate "Read src/server.py, switch the timeout retry to exponential backoff, and add a unit test"

# Read-only mode: analyze without modifying — good for a first pass on an unfamiliar repo
codemate --agent-mode readonly "Walk me through how a request flows through this project"

# Resume and share
codemate --resume last
codemate sessions list
codemate share <session-id>
```

### TUI keys

| Key | Action |
| --- | --- |
| `Enter` | Send message |
| `Esc` | Interrupt the current turn (completed tool results are kept) |
| `Ctrl+R` | Expand / collapse the full output of the latest tool call |
| `Ctrl+B` | Open the background-task panel (sub-agents and long-running commands) |
| `Ctrl+D` | Exit and save the session |
| `/` | Open the slash-command palette |

Common slash commands:

```
/model                 switch model
/compact               trigger one round of context compaction
/memory                inspect / edit the memory currently injected
/permissions           view and temporarily adjust permission rules
/agents                list sub-agent roles and their tool permissions
/cost                  token and cost breakdown for this session
/resume                switch to a previous session
/clear                 clear the context (memory is kept)
```

---

## Configuration

### Sources and precedence

Later sources override earlier ones:

1. Built-in defaults
2. Global config: `~/.codemate/config.yaml`
3. Project config: `<repo>/.codemate/config.yaml` (commit it — permission rules are worth sharing with the team)
4. Environment variables (`CODEMATE_*`)
5. CLI flags (`--model`, `--agent-mode`, …)

### Directory layout

```
~/.codemate/
├── config.yaml              # global config
├── memory/
│   ├── user.md              # cross-project user preferences
│   ├── feedback.md          # corrections and feedback
│   └── projects/<hash>.md   # project-level facts and conventions
├── sessions/
│   └── <session-id>.jsonl   # session traces for resume / share
└── offload/                 # L0 offload directory, keeps the restore path
```

### Full configuration example

```yaml
# ~/.codemate/config.yaml

model:
  provider: openai            # openai | anthropic | openai-compatible
  name: deepseek-chat
  base_url: https://api.deepseek.com/v1
  api_key_env: CODEMATE_API_KEY
  temperature: 0.2
  max_output_tokens: 8192
  # Cheaper model for auxiliary work: summarization, memory extraction
  small_model: deepseek-chat

context:
  max_tokens: 120000          # token ceiling that triggers compaction
  max_messages: 120           # message-count ceiling that triggers compaction
  trigger_ratio: 0.8          # start layered compaction at 80% of the ceiling
  l0_offload:                 # L0: offload large results, keep summary + restore path
    enabled: true
    threshold_tokens: 4000
    dir: ~/.codemate/offload
  l1_trim:                    # L1: trim and archive history
    enabled: true
    keep_recent_messages: 30
  l2_summarize:               # L2: model-generated summary
    enabled: true
    keep_recent_messages: 12
  prompt_too_long_recovery: true   # backstop: step-down retry when a request is oversized

subagent:
  enabled: true
  max_parallel: 3             # max concurrently running sub-agents
  background: true            # allow background execution with completion callbacks
  max_depth: 2                # no unbounded delegation chains
  roles:
    researcher:
      tools: [read, grep, glob, ls]                  # read-only
      model: deepseek-chat
    reviewer:
      tools: [read, grep, glob]                      # read-only
    tester:
      tools: [read, grep, bash]                      # executable
    coder:
      tools: [read, write, edit, grep, glob, bash]   # writable + executable

memory:
  enabled: true
  dir: ~/.codemate/memory
  scopes: [persistent, current_task]   # dual scope for write admission
  extract:
    enabled: true
    async: true               # extract after each turn without blocking the main loop
    dedup_similarity: 0.85    # similarity threshold for dedup
    conflict_strategy: merge  # merge | overwrite | keep_both
  inject:
    max_tokens: 2000          # injection budget
    mode: progressive         # index + relevance selection + read full text on demand

session:
  persist: true
  dir: ~/.codemate/sessions
  format: jsonl

permissions:
  default: ask                # deny | ask | allow
  rules:
    - { tool: read,  path: "**",              action: allow }
    - { tool: grep,  path: "**",              action: allow }
    - { tool: edit,  path: "src/**",          action: allow }
    - { tool: edit,  path: ".env*",           action: deny  }
    - { tool: write, path: "**",              action: ask   }
    - { tool: bash,  command: "git status",   action: allow }
    - { tool: bash,  command: "rm -rf *",     action: deny  }
  protected_paths:            # never writable by any tool
    - ~/.ssh/**
    - ~/.aws/**
    - .git/**
    - "**/*.pem"
  agent_mode: default         # default | readonly | plan | yolo

guardrails:
  read_before_edit: true      # a file must be read before it can be edited
  doom_loop:
    enabled: true
    window: 5                 # N consecutive near-identical tool calls counts as a loop
  tool_failure_budget:
    max_attempts: 3           # consecutive failures allowed per tool
    report_remaining: true    # write attempts_left / max_attempts back into the error
  request_budget:
    max_requests: 60          # model requests allowed per task
    on_exceed: interrupt      # convert to an Interrupt and ask whether to continue
  output_clipper:
    max_chars: 30000          # clip oversized tool output; L0 offload keeps the full text

tui:
  theme: dark                 # dark | light
  stream: true
  show_context_meter: true    # context usage and cost in the header
  collapse_tool_output: true
```

### Environment variables

| Variable | Description |
| --- | --- |
| `CODEMATE_API_KEY` | LLM API key |
| `CODEMATE_BASE_URL` | API endpoint, for OpenAI-compatible services |
| `CODEMATE_MODEL` | Overrides `model.name` |
| `CODEMATE_CONFIG` | Path to a specific config file |
| `CODEMATE_AGENT_MODE` | Overrides `permissions.agent_mode` |
| `CODEMATE_LOG_LEVEL` | `debug` / `info` / `warn` / `error` |

### Agent modes

| Mode | Permission filter | Use case |
| --- | --- | --- |
| `readonly` | Read tools only | Exploring an unfamiliar repo, code review |
| `plan` | Read-only plus plan output, nothing written to disk | Agree on an approach before touching code |
| `default` | Evaluated against `permissions.rules`; prompts on no match | Day-to-day development |
| `yolo` | Everything allowed (`protected_paths` still applies) | Sandboxes, containers, eval runs |

> Use `yolo` only in an isolated environment. Even there, `protected_paths` remains in force.

### Project-level configuration

Drop a `.codemate/config.yaml` in your repo root with only the fields you want to override, and commit the team's permission rules alongside the code:

```yaml
# <repo>/.codemate/config.yaml
permissions:
  rules:
    - { tool: bash, command: "pytest*",       action: allow }
    - { tool: bash, command: "make migrate*", action: ask   }
    - { tool: edit, path: "migrations/**",    action: deny  }

context:
  max_tokens: 160000
```

You can also add `.codemate/CODEMATE.md` describing the project's conventions (build commands, directory responsibilities, code style). It's loaded as the seed content for project-scoped memory.
