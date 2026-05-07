# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] - 2026-05-07

### Fixed

- **BUG-008 — `Could not set lock on file` on Ladybug DB no longer occurs** (reported by **uzuchi** on the v0.2.1 Zenn article).
  - **Before (v0.2.1)**: a separate OS-level process (`harness/hooks/cognee_remember_flusher.py`, scheduled by `crontab -e` or `nohup ... --daemon`) drained the queue by **spawning its own `cognee-mcp`**. The CLI helpers `src/sample_src/load_sample.py`, `src/sample_src/delete_sample.py`, and `src/knowledge_src/import_knowledge.py` did the same. While Claude Code was open it already held a `cognee-mcp`, so the second spawn hit Ladybug's non-blocking `fcntl(F_SETLK, F_WRLCK)` and failed immediately.
  - **After (v0.3.0)**: the queue is drained from **inside** the running Claude Code session by the new skill `harness/skills/cognee-queue-flush/SKILL.md`, scheduled by `/loop 5m cognee-queue-flush` (session-scoped) or `CronCreate(cron="*/5 * * * *", prompt="cognee-queue-flush", recurring=true, durable=true)` (persists across restarts via `~/.claude/scheduled_tasks.json`). The skill calls `mcp__cognee__remember` on the **already running** MCP cognee server, so no second `cognee-mcp` is ever created. The CLI helpers must now only be run while Claude Code is **not** running; for the delete case, `mcp__cognee__delete_dataset` from inside Claude Code is provided as a safe alternative. Verified with more than 50 MCP `remember` / `search` / `delete_dataset` calls under the BUG-008 reproduction condition: zero lock-contention errors.

- **BUG-009 — `mcp__cognee__remember` failures are no longer silently dropped from the queue.**
  - **Before (v0.2.1)**: cognee-mcp upstream returns failures with `is_error=False` and an `Error:`-prefixed text body. The v0.2.x flusher discarded the return value (commented-out `# result =`), so failed entries were marked as drained and lost.
  - **After (v0.3.0)**: the new skill enforces a 3-tier failure check (`is_error=True` / any `content[*].text` starting with `Error:` / any raised exception). On failure the entry is appended to `~/.claude/cognee_failed_remembers.jsonl` and kept in the queue, so the next firing retries it.

### Changed

- **Cognee 1.0.5 → 1.0.8.** Pin with `pip install "cognee[fastembed]==1.0.8"`. cognee 1.0.7 / 1.0.8 has an Ollama regression where `test_llm_connection` hits an Ollama URL without `/v1` and fails with 404; `config/.env.example` is pre-configured with `LLM_ENDPOINT=http://localhost:11434/v1` (the `/v1` is required) and `COGNEE_SKIP_CONNECTION_TEST=true` to work around it.
- **The `cognee-queue-flush` skill now lets you cap how many queue entries it processes per firing via the environment variable `COGNEE_QUEUE_FLUSH_MAX_PER_RUN` (default `3`).** Each `mcp__cognee__remember` call takes seconds to tens of seconds depending on your LLM and machine, and a drain that exceeds the schedule interval would overlap with the next firing. Tune the env var to your environment; `docs/HARNESS_GUIDE.md` Step 4 and `docs/SETUP.md` §2-4 list rough starting points (cloud LLM 10–20, qwen2.5:14b on GPU 3–5, CPU only 1–2, smaller local models 5–10).
- **`docs/HARNESS_GUIDE.md` and `docs/SETUP.md` rewritten** for the new architecture: install the skill in Step 1, schedule it via `/loop` or `CronCreate` in Step 4, and a "Lifetime" column makes the session-scoped vs. `durable=true` difference explicit. `harness/settings.example.json` no longer references the flusher hooks or their Bash permissions.

### Removed

- `harness/hooks/cognee_remember_flusher.py` and the OS-level `crontab -e` / `nohup --daemon` paths that scheduled it. See **Migration** below.

### Migration (from v0.2.1)

1. Pull v0.3.0.
2. Re-run Step 1 of `docs/HARNESS_GUIDE.md` (re-copy the hooks and copy the new `harness/skills/cognee-queue-flush` directory into `~/.claude/skills/`).
3. Delete `~/.claude/hooks/cognee_remember_flusher.py` and the matching `*/5 * * * * .../cognee_remember_flusher.py` line from `crontab -e`.
4. Restart Claude Code so it picks up the new skill.
5. Inside the new Claude Code session, register the schedule once: type `/loop 5m cognee-queue-flush`, or — to keep the schedule across restarts — ask Claude Code to call `CronCreate(cron="*/5 * * * *", prompt="cognee-queue-flush", recurring=true, durable=true)` (only the AI-side `CronCreate` Tool exposes `durable=true`; the `/loop` slash command does not).

### Known Issues

- **`save_interaction` is still unavailable** (BUG-007). **cognee-mcp 0.5.4 calls cognee 1.0.8's `add_rule_associations` function with the keyword argument `context=...`, but cognee has already renamed that argument to `ctx=...`, so the call fails with a keyword-argument mismatch.** Use `remember` for immediate persistence of interaction text. Tracked in the upstream cognee-mcp project.

### Special Thanks

- **uzuchi** — for the Zenn comment on v0.2.1 reporting `Could not set lock on file` together with the exact reproduction setup (Windows-native Claude Code → `wsl.exe -d Ubuntu-24.04 -- python3 .../start_cognee_mcp.py` over stdio transport, `shared_ladybug_lock` unset, Redis not running). That report is what made this rework possible. Thank you.

## [0.2.1] - 2026-05-04

### Changed
- Code style cleanup only — **no behavior changes**. v0.2.0 verification
  results (UT 110 / IT 6 / ET 4 / ST 4 all passed; qwen2.5:14b matrix
  verification 8 tools × 5 runs) remain valid because no logic was modified.
- Applied Ruff lint fixes to comply with cognee-integrations coding
  standards (`line-length = 100`, `select = ["E", "F", "I", "W"]`,
  `target-version = "py310"`):
  - `harness/hooks/auto_remember_completion.py`: F401 — unused
    `import os` commented out and moved out of import block.
  - `harness/hooks/auto_remember_user_message.py`: F401 — unused
    `import os` and `import subprocess` commented out and moved out
    of import block.
  - `harness/hooks/cognee_remember_flusher.py`: F841 — `result =`
    assignment commented out (function call retained); E501 — argparse
    `--interval` line wrapped to fit 100-char limit.
  - `src/knowledge_src/import_knowledge.py`: E501 — five logger and
    argparse lines wrapped (one log message wording slightly shortened
    while preserving meaning: "the failure" → "failure", "the cause"
    → "cause").
  - `src/main_src/import_to_graph.py`: I001 — `urllib.error` and
    `urllib.request` imports reordered alphabetically; E501 — argparse
    description and three help lines wrapped (one description shortened
    while preserving meaning: "production runtime" → "runtime").

### Why
- Preparing the toolkit for potential cognee-integrations contribution
  (per cognee co-founder Vasilije Markovic's invitation on X to send a
  PR with toolkit features). The cognee-integrations CI enforces
  Ruff lint rules above; passing those rules ahead of time avoids
  CI rejections during the PR review process.

## [0.2.0] - 2026-05-04

### Added
- New `knowledge/sample_knowledge/05_graph_memory_operations.md`: a sample
  ingestion file documenting operational know-how for Cognee graph memory
  (when to call cognify, remember vs. save_interaction, search-type choice,
  recall auto-routing, and forget_memory/improve/prune usage). The bundled
  sample count therefore increased from 4 to 5 files.

### Changed
- **Upgraded Cognee from 1.0.3 to 1.0.5**. Cognee 1.0.4 replaced the embedded graph
  database from KuzuDB to **Ladybug DB**, and this distribution follows that change.
  - Dependency: `kuzu==0.11.3` → `ladybug==0.16.0` (auto-replaced; backward-compatible aliases retained)
  - Verified Cognee version: 1.0.5
  - cognee-mcp version: 0.5.4 (no change)
- Updated documentation to replace **"KuzuDB" with "Ladybug DB"** throughout:
  - `README.md` (features and tech stack table)
  - `config/.env.example` (COGNEE_DATA_PATH comments and storage description)
  - `docs/SETUP.md` (verified versions and pinning examples)
  - `docs/GETTING_STARTED.md` (verified Cognee version and measured response times)
- Updated the bundled-sample file-count notation from 4 to 5:
  - `README.md` (directory structure section)
  - `docs/GETTING_STARTED.md` (including the `[N/M]` ingestion log examples)
- Reorganized design-decision notes in
  `knowledge/sample_knowledge/03_design_decisions.md`. KuzuDB/LanceDB/FastEmbed
  are bundled with Cognee and used automatically, so they are not user-side
  selections. The text was consolidated under "Why Cognee was adopted" and
  notes that the graph DB is Ladybug DB in v0.2.0 / KuzuDB in v0.1.x.
- Updated error-handling notes in
  `knowledge/sample_knowledge/04_common_errors.md` so that local-LLM examples
  use `qwen2.5:14b` (the only locally verified LLM for v0.2.0) instead of
  `llama3.1:8b`.
- Updated the recall failure-condition note in
  `harness/rules/cognee_memory_usage.md` from "may fail when running on
  `llama3.1:8b`" to "may fail when running on local LLMs other than
  `qwen2.5:14b`".

### Fixed (pre-existing source code defects from v0.1.10–v0.1.12)
- `src/main_src/import_to_graph.py`:
  - Removed a stale display line in `list_targets()` for the `comments` target
    that was not defined in `TARGET_MAP` (a leftover from v0.1.10).
  - `check_ollama()` now runs only when `LLM_PROVIDER=ollama`
    (previously it failed unnecessarily when a cloud API was configured).
  - `check_ollama()` is now skipped during `--dry-run`
    (the dry-run mode only prints the file list and does not need Ollama).
- `src/knowledge_src/import_knowledge.py`:
  - Added the same `LLM_PROVIDER=ollama` guard to `check_ollama()`.
- `harness/hooks/auto_remember_user_message.py`:
  - Removed contradictory docstring lines that claimed "this sample provides the
    simpler direct-call variant" while the actual implementation uses the queue
    approach exclusively.
- `harness/hooks/cognee_remember_flusher.py`:
  - Fixed the `remaining` filter from `not line.strip()` to `line.strip()`
    (an internal bug that caused failed entries to be silently dropped from
    the queue; the data itself was still preserved in `failed.jsonl`, but
    the queue — the source of retry attempts — held the wrong contents).
  - Removed an unnecessary `sys.path.insert(main_src)` in `remember_via_mcp()`
    (fastmcp is importable directly from the distribution's venv site-packages).

### Verified (in v0.2.0)
- **qwen2.5:14b (num_ctx=8192) × Ladybug DB: 35/40 ✅** verified
  - remember 5/5 ✅, search(CHUNKS) 5/5 ✅, search(GRAPH_COMPLETION) 5/5 ✅, recall 5/5 ✅
  - cognify 5/5 ✅, improve 5/5 ✅, forget_memory 5/5 ✅
  - **save_interaction 0/5 ❌** (known limitation, see below)
- **Ladybug DB (introduced in Cognee 1.0.4) accelerates graph traversal**, making
  qwen2.5:14b practically usable (significant subjective improvement over the
  v0.1.x KuzuDB environment).
  - search(CHUNKS): avg 3.2s (deterministic, no LLM)
  - search(GRAPH_COMPLETION): avg 14.6s (range 12-18s)
  - recall (Q-A, TEMPORAL routing): 20-24s
  - recall (Q-B, GRAPH_COMPLETION_COT routing): 154-156s
  - improve / forget_memory: all immediate (under a few seconds)

### Known Issues
- **save_interaction is unavailable** (API mismatch between cognee-mcp 0.5.4 and cognee 1.0.5)
  - Error: `add_rule_associations() got an unexpected keyword argument 'context'`
  - Cause: cognee 1.0.5 renamed the `add_rule_associations` argument from `context` to `ctx`,
    but cognee-mcp 0.5.4 has not been updated (upstream `topoteretes/cognee` main branch is
    in the same state)
  - Workaround: Use `remember` for immediate persistence of interaction text

### Migration (from v0.1.x)
- Existing graph DB data (KuzuDB) from Cognee 1.0.3 is automatically migrated to Ladybug
  format on first startup of Cognee 1.0.4+.
- To preserve existing data: `pip install -U "cognee[fastembed]==1.0.5"` and start Cognee
  once → automatic migration runs.
- To reset data: run `forget_memory(everything=True)` to clear all data, then re-cognify
  in the Cognee 1.0.5 environment.

## [0.1.12] - 2026-05-03

### Fixed
- Hardware notation in `README.md` and `docs/GETTING_STARTED.md` is now
  expressed in terms of **VRAM capacity**, not GPU model name. Previous
  text said "RTX 4070 12GB or higher", but the laptop variant of the
  RTX 4070 has only 8GB of VRAM, which would mislead users into
  thinking their laptop met the requirement when it does not. The
  recommended threshold is now stated as "GPU with 12GB+ VRAM" with a
  note about laptop variants. The verified-minimum entry is now
  "NVIDIA GeForce RTX 4060 Laptop GPU (VRAM 8GB)" so the actual tested
  device is named precisely.
- `CHANGELOG.md` has been trimmed to keep only public-relevant
  entries (v0.1.10 onward); the pre-public v0.1.0..v0.1.9 history was
  internal development noise.

## [0.1.11] - 2026-05-02

### Changed
- Switched the default local LLM in `config/.env.example` to
  `qwen2.5:14b` (num_ctx=8192). Setup examples for Claude API and
  OpenAI API are also included as comments.
- Added a "Recommended LLM and Environment" section to
  `docs/GETTING_STARTED.md`.
  - Cloud APIs (Claude / OpenAI) are **strongly recommended** thanks
    to their official structured-output support.
  - Local LLM operation requires a GPU with **12GB+ VRAM** and
    **qwen2.5:32b or larger** (14B is the practical minimum).
- `src/main_src/import_to_graph.py` and
  `src/knowledge_src/import_knowledge.py` now read `LLM_MODEL` from
  `config/.env` dynamically (the previous `llama3.1:8b` hardcoding has
  been removed).
- Replaced `llama3.1:8b`-specific text in `docs/GETTING_STARTED.md`
  with `qwen2.5:14b` / cloud-API guidance (sample-ingestion failure
  fallback list, `recall` fallback notice, and the troubleshooting
  section).

## [0.1.10] - 2026-05-02

### Added
- Initial public release. A module that adds Cognee-based graph memory
  to Claude Code.
  - `src/main_src/start_cognee_mcp.py` — MCP server startup script
  - `src/main_src/import_to_graph.py` — production ingestion
  - `src/sample_src/load_sample.py` / `delete_sample.py` — bundled
    sample handling
  - `src/knowledge_src/split_knowledge.py` / `import_knowledge.py` —
    user-knowledge ingestion pipeline
  - `docs/SETUP.md` / `docs/GETTING_STARTED.md` /
    `docs/HARNESS_GUIDE.md` — setup, usage, and harness installation
    guides
  - `knowledge/sample_knowledge/` — 4 bundled sample files for
    end-to-end verification
  - `harness/` — Claude Code × Cognee auto-accumulation harness
    (optional)
  - Fully local operation possible (Ollama + FastEmbed, no external
    API keys required)
