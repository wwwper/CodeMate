<div align="center">

# CodeMate

**A lightweight local coding agent built for real codebases**

Long-context management · Sub-agent collaboration · Persistent memory · Permission & reliability guardrails


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

---

## Quick start

### Requirements

- Python **3.10+**
- Git
- An LLM API key (OpenAI / Anthropic / DeepSeek / any OpenAI-compatible endpoint)

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
