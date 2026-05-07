# Environment Setup Guide

---

## 1. Prerequisites

### 1-1. Verified environment

| Item | Value |
|------|---|
| OS | Linux (Ubuntu 22.04 or later) / WSL2 |
| Python | 3.12 or higher |
| Ollama | Latest version (https://ollama.com) |
| LLM model | qwen2.5:14b (`ollama pull qwen2.5:14b`) |
| Claude Code | Latest version |

### 1-2. Ollama startup check

```bash
ollama serve             # Start in background
ollama list              # Show available models
ollama pull qwen2.5:14b  # Download model if not yet pulled
```

---

## 2. Setup steps

### 2-1. Overview

| Step | Action | Time estimate |
|---------|------|------------|
| 1 | Clone the repository | 1 min |
| 2 | Create `.env` and configure paths | 2 min |
| 3 | Build venv and `pip install` | 5–15 min (includes FastEmbed model download) |
| 4 | Add execute permission to startup script | 1 min |
| 5 | Register MCP with Claude Code | 1 min |

### 2-2. Step details

**Step 1: Clone the repository**
```bash
git clone https://github.com/JapanNomu/tools.git
cd tools/claude-code-tools/claude-code-cognee-graph-memory
```

**Step 2: Create `.env` and configure paths**

```bash
cp config/.env.example config/.env
```

Open `config/.env` and replace the following two values with absolute paths in your environment.

**(1) How to obtain the absolute path**

Run `pwd` inside the cloned directory; it prints the absolute path.

```bash
pwd
# Example output: /home/yourname/tools/claude-code-tools/claude-code-cognee-graph-memory
```

**(2) Values to configure in `config/.env`**

Use the absolute path obtained from `pwd` to fill in the following.

| Setting | Example value |
|---------|-----------|
| `SYSTEM_ROOT_DIRECTORY` | `<pwd output>/data/cognee/system` |
| `DATA_ROOT_DIRECTORY` | `<pwd output>/data/cognee/data` |

For example, if `pwd` returned `/home/yourname/tools/claude-code-tools/claude-code-cognee-graph-memory`:

```bash
SYSTEM_ROOT_DIRECTORY=/home/yourname/tools/claude-code-tools/claude-code-cognee-graph-memory/data/cognee/system
DATA_ROOT_DIRECTORY=/home/yourname/tools/claude-code-tools/claude-code-cognee-graph-memory/data/cognee/data
```

**(3) Notes**

- Relative paths (e.g. `./data/cognee/...`) are not supported. **Always use absolute paths (starting with `/`)**.
- The `data/cognee/system` and `data/cognee/data` directories are created automatically (no manual creation required).
- `COGNEE_DATA_PATH` can be left at its default value (`./data/cognee`) unless you need to change it.

**Step 3: Build venv**
```bash
cd src
python3 -m venv venv
source venv/bin/activate
pip install cognee-mcp "cognee[fastembed]"
deactivate
cd ..
```

**Step 4: Add execute permission**
```bash
chmod +x src/main_src/start_cognee_mcp.py
```

**Step 5: Register MCP**

Register the script with an **absolute path** to the location you cloned to. Reuse the `pwd` output from Step 2.

For example, if `pwd` returned `/home/yourname/tools/claude-code-tools/claude-code-cognee-graph-memory`, run:

```bash
claude mcp add cognee --scope user /home/yourname/tools/claude-code-tools/claude-code-cognee-graph-memory/src/main_src/start_cognee_mcp.py
claude mcp list  # Setup is complete when "cognee" shows ✓ Connected
```

Replace the path above with **your own absolute path** (the `pwd` output from Step 2 + `/src/main_src/start_cognee_mcp.py`).

> **Why an absolute path is required:** `--scope user` registers the server globally for all your projects. If you register it with a relative path (e.g. `src/main_src/start_cognee_mcp.py`), Claude Code resolves the path against the current working directory at launch time — so it only connects when you start `claude` from this cloned directory and fails everywhere else.

### 2-3. Verifying the setup

**Important: Restart Claude Code if it is already running**

The settings registered with `claude mcp add` are **not picked up by Claude Code sessions that are already running**. Restart your session as follows:

| Environment | How to restart |
|---|---|
| VSCode extension | Command Palette (`Ctrl+Shift+P`) → `Developer: Reload Window` |
| Terminal `claude` command | Exit the session (`Ctrl+D` or `/exit`) and start `claude` again |

**Three ways to verify the connection**

- Terminal: run `claude mcp list` and check that `cognee: ✓ Connected` is shown
- VSCode: open the "MCP servers" panel and check that `cognee: ✓ Connected` is shown
- Inside a session: run `/mcp` and check that `cognee: connected` is shown

### 2-4. Schedule the queue-flush batch processor (v0.3.0)

The hooks (`auto_remember_user_message.py` / `auto_remember_completion.py`) only append to a queue file (`~/.claude/cognee_pending_remembers.jsonl`). To actually persist the queued entries into Cognee graph memory, the `cognee-queue-flush` skill must run periodically inside the Claude Code session.

**Architecture rationale**: This skill runs in the same Claude Code process ("live in the current process") and reuses the existing MCP cognee server. It does NOT spawn a new cognee-mcp process. This is what avoids BUG-008 (Ladybug DB lock contention).

**Setup**: Inside Claude Code, register the skill on a recurring schedule. Choose one of:

| Method | Command | Lifetime |
|---|---|---|
| `/loop` (interactive) | `/loop 5m cognee-queue-flush` | **Session only** — dies when you exit Claude Code |
| `CronCreate` (in-session) | Inside Claude Code: `CronCreate(cron="*/5 * * * *", prompt="cognee-queue-flush", recurring=true)` | **Session only** — dies when you exit Claude Code |
| `CronCreate(durable=true)` (persistent) | Inside Claude Code: `CronCreate(cron="*/5 * * * *", prompt="cognee-queue-flush", recurring=true, durable=true)` | **Persistent** — saved to `~/.claude/scheduled_tasks.json`, auto-restored on next launch |

The recommended cadence is **every 5 minutes**. Tune to your needs.

**Persistence note**: `/loop` and `CronCreate(durable=false)` registrations live inside one Claude Code session only — you must re-register them each time you launch Claude Code. Only `CronCreate(durable=true)` survives restarts.

**`/loop` cannot be made persistent** — the slash command does not expose a `durable` option. The `durable=true` form is only available through the `CronCreate` Tool, which is invoked by Claude Code (the AI), **not typed by you**. To set up a persistent schedule, ask Claude Code in chat to run it once, e.g.:

> Please call `CronCreate` with `cron="*/5 * * * *"`, `prompt="cognee-queue-flush"`, `recurring=true`, `durable=true` so the queue keeps draining after restarts.

The AI runs the tool call once, the schedule is saved, and you do not need to ask again.

See `docs/HARNESS_GUIDE.md` Step 4 for the full discussion (per-invocation drain limit tuning via `COGNEE_QUEUE_FLUSH_MAX_PER_RUN`, persistence trade-offs, etc.).

**Verify**: After setting up, send a message to Claude Code, wait 5 minutes, then run inside Claude Code: `mcp__cognee__search(search_query="<some recent text>", search_type="CHUNKS")`. The recent text should be retrievable.

### 2-5. CLI tools usage constraint (v0.3.0)

The following CLI tools spawn a new `cognee-mcp` process and **must NOT be run while Claude Code is open** (it will trigger BUG-008 lock contention):

- `src/sample_src/load_sample.py` (load bundled samples)
- `src/sample_src/delete_sample.py` (delete sample dataset)
- `src/knowledge_src/import_knowledge.py` (bulk-import your own knowledge files)

**Usage rule**: Run these tools **before launching Claude Code**, or after exiting Claude Code.

**Alternative for delete**: To delete a dataset while Claude Code is running, call `mcp__cognee__delete_dataset` from within Claude Code instead of running the CLI script.

---

## 3. Dependency version policy

| Item | Description |
|------|------|
| Install method | `pip install cognee-mcp "cognee[fastembed]"` (latest version) |
| Pinned version file | `src/requirements.txt` (not yet provided; future work) |
| Verified versions | Cognee 1.0.8, cognee-mcp 0.5.4, ladybug (bundled with cognee 1.0.8); all 21 BATCH tests passing (as of 2026-05-07, v0.3.0) |
| Pinning specific versions | Use e.g. `pip install "cognee-mcp==0.5.4" "cognee[fastembed]==1.0.8"` |
| Required when using Ollama | cognee 1.0.7/1.0.8 has a regression where `test_llm_connection` hits an Ollama URL without `/v1` and fails with 404. Set `LLM_ENDPOINT=http://localhost:11434/v1` (the `/v1` is required) and `COGNEE_SKIP_CONNECTION_TEST=true` in `config/.env`. Both are pre-configured in `.env.example`. |

---

## 4. Settings to update when relocating

| Target | Change | Required/Optional |
|---------|---------|---------|
| `config/.env` `SYSTEM_ROOT_DIRECTORY` | Absolute path in your environment | **Required** |
| `config/.env` `DATA_ROOT_DIRECTORY` | Absolute path in your environment | **Required** |
| `config/.env` `COGNEE_DATA_PATH` | Leave at default (`./data/cognee`) unless you need to change it | Optional |
| `config/.env` `LLM_MODEL` | When using a different Ollama model | Optional |
| `config/.env` `LLM_ENDPOINT` | When Ollama runs on a different host | Optional |
| `config/.env` `EMBEDDING_MODEL` / `EMBEDDING_DIMENSIONS` | When using a different embedding model | Optional |

**No change needed:** `src/main_src/start_cognee_mcp.py` (resolves the project root automatically via `Path(__file__)`)
