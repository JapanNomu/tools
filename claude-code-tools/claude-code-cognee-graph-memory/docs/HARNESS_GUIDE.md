# HARNESS_GUIDE — Claude Code × Cognee Auto-Accumulation Harness

## [Claude Code × Cognee — Practical Know-How Accumulation Tool]

**RAG gives you the answer. Cognee gives you the whole story.**
Claude Code remembers why decisions were made.
**Never say the same thing twice again.**

## What is this harness?

**A mechanism that accumulates your own, personal know-how into Cognee graph memory
the more you use Claude Code.**

- User messages and AI responses are automatically recorded into the graph by hooks
- From the next session onward, the AI is required to search graph memory before
  starting any work
- The same mistakes are not repeated; past decisions, context, and related facts
  come back to you in a connected chain

Plain vector search (RAG) returns only the matching chunk. With graph structure on
top, Cognee returns the rationale, the timeline, and related facts as well — so you
can recover **why** something was decided, **when**, and **what else relates to it**.

---

## How it works

```
┌────────────────────────────────────────────────────────┐
│ Claude Code session                                     │
│                                                         │
│  User message ─────► UserPromptSubmit hook ────┐       │
│                                                ▼        │
│                                  ~/.claude/             │
│                                  cognee_pending_        │
│                                  remembers.jsonl        │
│                                  (queue)                │
│                                                ▲        │
│  AI response done ───► Stop hook ──────────────┘        │
│                                                         │
│  AI runs                                                │
│  search(CHUNKS) ◄─── enforced via CLAUDE.md / rules     │
└────────────────────────┬────────────────────────────────┘
                         │
                         │ cognee-queue-flush skill drains the queue periodically
                         │ (scheduled via /loop or CronCreate inside Claude Code,
                         │  reuses the existing MCP cognee server — no new process)
                         ▼
                ┌─────────────────────┐
                │ Cognee graph memory │
                │ (persistent)        │
                └─────────────────────┘
```

---

## Installation (5 steps)

### Prerequisites

- The base distribution setup (`docs/SETUP.md`) is complete
- `claude mcp list` shows `cognee` as registered
- `src/sample_src/load_sample.py` has run successfully

### Step 1: Copy harness files into ~/.claude/

```bash
# Run from the distribution root

# Rule (Cognee usage policy)
cp harness/rules/cognee_memory_usage.md ~/.claude/rules/

# Hooks (queue writers only — v0.3.0 has no flusher.py)
cp harness/hooks/auto_remember_user_message.py ~/.claude/hooks/
cp harness/hooks/auto_remember_completion.py ~/.claude/hooks/
chmod +x ~/.claude/hooks/auto_remember_user_message.py
chmod +x ~/.claude/hooks/auto_remember_completion.py

# Skill (queue drainer — replaces v0.2.x flusher.py / required for Step 4)
mkdir -p ~/.claude/skills
cp -r harness/skills/cognee-queue-flush ~/.claude/skills/
```

> **v0.3.0 architecture note**: in v0.2.x the queue was drained by an OS-level
> `cognee_remember_flusher.py` running under cron. v0.3.0 replaces it with the
> `cognee-queue-flush` skill, which runs **inside the existing Claude Code
> session** and reuses the existing MCP `cognee` server — it does not spawn a
> new `cognee-mcp` process, which is what avoids the Ladybug DB lock contention
> error (`Could not set lock on file`).

### Step 2: Merge ~/.claude/settings.json

Manually merge the contents of `harness/settings.example.json` into your existing
`~/.claude/settings.json`.

**Important keys to merge**:
- `hooks.UserPromptSubmit` — record user messages
- `hooks.Stop` — record AI response summaries
- `permissions.allow` — auto-approve Cognee MCP tools (`search`, `remember`, etc.)

Append to the existing `hooks` and `permissions` rather than overwriting them.

### Step 3: Append the snippet to your project's CLAUDE.md

Append the contents of `harness/CLAUDE_md_sample.md` to the end of the `CLAUDE.md`
of any project where you want the harness to be active.

This ensures that the AI **always searches Cognee graph memory before starting
any work** in that project.

### Step 4: Schedule the cognee-queue-flush skill

The hooks only append to a queue file; the `cognee-queue-flush` skill actually
performs the `remember` calls (via the existing MCP cognee server — no new
process is spawned).

Choose one of:

| Method | Command (run inside Claude Code) | When to use |
|---|---|---|
| `/loop` (interactive) | `/loop 5m cognee-queue-flush` | Development / testing — easy to start/stop, dies on session exit |
| `CronCreate` (persistent within session) | `CronCreate(cron="*/5 * * * *", prompt="cognee-queue-flush", recurring=true)` | Always-on use — survives within the session for up to 7 days |

> **Why no OS-level cron in v0.3.0**: an OS-level cron job would spawn a new
> `cognee-mcp` process every interval, conflicting with the MCP `cognee` server
> already held by your Claude Code session and triggering the Ladybug DB lock
> contention error (`Could not set lock on file`). The Claude Code-internal
> scheduler keeps everything in one process, which is required by design.

For the same reason, the CLI helpers `src/sample_src/load_sample.py` and
`src/sample_src/delete_sample.py` must only be run when **Claude Code is not
running** (see SETUP.md §2-5).

#### Tuning the per-invocation drain limit

By default, each invocation of the `cognee-queue-flush` skill processes at most
**3 entries** from the queue and leaves any remainder for the next firing. This
default is intentionally conservative — **how many entries your machine can
process within one schedule interval depends on your environment** (CPU/GPU,
VRAM, the LLM you chose, network latency for cloud LLMs).

Tune the cap with the environment variable `COGNEE_QUEUE_FLUSH_MAX_PER_RUN`:

```bash
# Example: process up to 10 entries per invocation (cloud LLM, fast machine)
export COGNEE_QUEUE_FLUSH_MAX_PER_RUN=10
```

Add this `export` to your shell profile (`~/.bashrc`, `~/.zshrc`, etc.) so
Claude Code inherits it on every launch.

**Rough starting points** (always tune to your own environment by observing
how long one invocation takes — the skill reports `(succeeded, failed, remaining)`
and you can watch the queue drain over a few cycles):

| Setup | Suggested value |
|---|---|
| Cloud LLM (Claude / OpenAI / Gemini) | 10 — 20 |
| Local Ollama, qwen2.5:14b on a discrete GPU (8GB+ VRAM) | 3 — 5 (default 3) |
| Local Ollama, qwen2.5:14b on CPU only | 1 — 2 |
| Local Ollama, smaller models (qwen2.5:7b etc) | 5 — 10 |

If you set the value too high for your machine, one drain may exceed the
schedule interval (default 5 min) and overlap with the next firing. If you
set it too low, the queue may grow faster than it drains during heavy use.
Start at the default and adjust based on observed behaviour.

#### Persistence across Claude Code restarts

`/loop` and `CronCreate` registrations live **inside one Claude Code session
only**. When you exit Claude Code (or `Reload Window` in VSCode), the schedule
disappears and the queue stops draining until you re-register it.

To make the schedule survive restarts you must use `CronCreate` with
`durable=true`:

```
CronCreate(
    cron="*/5 * * * *",
    prompt="cognee-queue-flush",
    recurring=true,
    durable=true,   # <-- key difference
)
```

With `durable=true`, the registration is written to
`~/.claude/scheduled_tasks.json` and Claude Code restores it automatically on
the next launch — no re-registration needed.

> ⚠️ **`/loop` cannot make a schedule persistent.** The slash command does not
> expose a `durable` option. Persistence is only available through the
> `CronCreate` Tool, which is **invoked by Claude Code (the AI), not typed by
> you**. If you want a persistent schedule, ask Claude Code in chat to do this
> once, e.g.:
>
> > Please call `CronCreate` with `cron="*/5 * * * *"`,
> > `prompt="cognee-queue-flush"`, `recurring=true`, `durable=true` so the
> > queue keeps draining after restarts.
>
> The AI runs the tool call once, the schedule lands in
> `~/.claude/scheduled_tasks.json`, and you do not need to ask again.

If you only ever use one session at a time and re-launching Claude Code is
fine for you, the in-session `/loop 5m cognee-queue-flush` is simpler — just
remember to type it once per session.

### Step 5: Restart Claude Code

Changes to `settings.json` and to the hook files are not picked up by an already
running Claude Code session.

- VSCode Claude Code extension: `Reload Window`
- Terminal `claude` command: exit and re-launch

Verify the connection:
```bash
claude mcp list
# cognee should appear with ✓ Connected
```

---

## Verifying the harness

### Is the harness recording?

1. Send any message to Claude Code
2. `cat ~/.claude/cognee_pending_remembers.jsonl` — your message should appear
3. Trigger the skill manually inside Claude Code: invoke the `cognee-queue-flush`
   skill (e.g. `/cognee-queue-flush` or via the Skill tool)
4. The skill prints a summary at the end (succeeded_count / failed_count) and
   `~/.claude/cognee_pending_remembers.jsonl` should be drained (empty or shorter)
5. From Claude Code, run `search("a keyword from your message", search_type="CHUNKS")`
   — the message should appear in the results

### Is the AI calling search?

Compare the AI's behaviour for the same kind of task before and after enabling the
harness:

- Before: jumps straight into Edit / Bash
- After: first calls `mcp__cognee__search(...)` and only then proceeds

If the after-behaviour does not happen, the snippet may not have made it into your
project's CLAUDE.md.

---

## Troubleshooting

### Hooks aren't running (queue file is not updated)

- Verify the structure of `hooks.UserPromptSubmit` / `hooks.Stop` in `~/.claude/settings.json`
- Did you restart Claude Code? Hook changes require a restart
- Try running `python3 ~/.claude/hooks/auto_remember_user_message.py < /dev/null` standalone

### Skill fails

- Check the skill's summary printed in the Claude Code chat
  (succeeded_count / failed_count and any error messages)
- Failed entries are kept in `~/.claude/cognee_failed_remembers.jsonl` for inspection
- Confirm the skill is installed: `ls ~/.claude/skills/cognee-queue-flush/SKILL.md`
- Confirm Claude Code can see the skill: a fresh session may be required after
  copying the skill into `~/.claude/skills/`
- If the project root cannot be located, set `COGNEE_GRAPH_MEMORY_ROOT`:
  ```bash
  export COGNEE_GRAPH_MEMORY_ROOT=/path/to/claude-code-cognee-graph-memory
  ```

### The AI doesn't call search

- Confirm the `harness/CLAUDE_md_sample.md` content is in your project's `CLAUDE.md`
- `CLAUDE.md` is read every turn, so a Claude Code restart is not required, but a
  new session may be needed to pick up the change
- For a stronger rule, place `harness/rules/cognee_memory_usage.md` under
  `~/.claude/rules/`

### The queue is filling up faster than it drains

- The `cognee-queue-flush` skill is probably not scheduled or not firing.
  Confirm `/loop 5m cognee-queue-flush` is active in this session, or that
  `CronCreate(cron="*/5 * * * *", ...)` was registered
- Each entry takes a few seconds to a few tens of seconds to ingest, so the queue
  may be populated on first start
- Drain manually: invoke the `cognee-queue-flush` skill once from inside Claude Code

---

## Files in this harness

| File | Purpose |
|---|---|
| `harness/CLAUDE_md_sample.md` | Snippet to append to a project's CLAUDE.md |
| `harness/rules/cognee_memory_usage.md` | Long-form rule for `~/.claude/rules/` |
| `harness/hooks/auto_remember_user_message.py` | UserPromptSubmit hook (queue user messages) |
| `harness/hooks/auto_remember_completion.py` | Stop hook (queue AI response summaries) |
| `harness/skills/cognee-queue-flush/SKILL.md` | Skill that drains the queue via the existing MCP cognee server (replaces v0.2.x flusher.py) |
| `harness/settings.example.json` | Example for merging into ~/.claude/settings.json |

---

## Outcome (the longer you use it, the more you accumulate)

- After 1 day: dozens of entries
- After 1 week: hundreds of entries (you can already retrieve past corrections and decisions)
- After 1 month: thousands of entries — your own personal AI knowledge base

**The more you use Claude Code, the more your AI grows into yours.**
