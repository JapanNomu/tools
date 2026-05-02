# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.10] - 2026-05-02

### Changed
- Cleaned up the repository history. The earlier v0.1.0 through v0.1.9
  history carried internal project identifiers and intermediate
  file-organisation traces, so the GitHub repository was re-created and
  v0.1.10 is published as the **clean, public-facing initial release**.
  The contents and structure of this distribution are equivalent to
  v0.1.9 — **only the git history has been replaced**.
- A summary of the changes that lived in the old history is preserved
  in this CHANGELOG under the `[0.1.0]`–`[0.1.9]` entries (kept for
  transparency).

## [0.1.9] - 2026-05-02

### Changed
- Translated all remaining Japanese comments and docstrings inside
  `harness/hooks/` and `harness/settings.example.json` into English so
  that the English distribution is fully consistent in language. The
  files that were updated:
  - `harness/hooks/auto_remember_user_message.py`
  - `harness/hooks/auto_remember_completion.py`
  - `harness/hooks/cognee_remember_flusher.py`
  - `harness/settings.example.json`
- This release is published in parallel with the Japanese-language
  distribution `JapanNomu/tools-ja` so that each language can be
  consumed in its native form, end-to-end (docs, code comments,
  configuration comments).

## [0.1.8] - 2026-05-02

### Added
- New `harness/` directory: a Claude Code × Cognee auto-accumulation harness
  that records user messages and AI responses into Cognee graph memory and
  enforces a "search before any work" rule via CLAUDE.md / `~/.claude/rules/`.
  The harness ships with:
  - `harness/CLAUDE_md_sample.md` — snippet to append to a project's CLAUDE.md
  - `harness/rules/cognee_memory_usage.md` — long-form rule for `~/.claude/rules/`
  - `harness/hooks/auto_remember_user_message.py` — UserPromptSubmit hook
  - `harness/hooks/auto_remember_completion.py` — Stop hook
  - `harness/hooks/cognee_remember_flusher.py` — queue drainer (cron recommended)
  - `harness/settings.example.json` — settings.json merge example
- New `docs/HARNESS_GUIDE.md`: end-to-end installation guide for the harness.
- `README.md` is updated to document the harness and the catchphrase
  ("RAG gives you the answer. Cognee gives you the whole story.").

## [0.1.7] - 2026-05-02

### Added
- `docs/GETTING_STARTED.md`: added "fallback to `search(CHUNKS)`"
  notices next to `recall` examples in three places (Scenario A,
  Step 3, and the tool reference table). With llama3.1:8b, `recall`
  may fail with an "LLM format error" because the model does not
  always emit the JSON shape Cognee expects. The notice now points
  users directly to the existing fallback in the Troubleshooting
  section so they can recover without leaving the page.

## [0.1.6] - 2026-05-02

### Added
- `docs/SETUP.md` Section 2-3 ("Verifying the setup"): added an
  "Important: Restart Claude Code if it is already running" notice.
  Settings registered with `claude mcp add` are not picked up by
  sessions that are already running, so users with an active VSCode
  Claude Code extension or terminal `claude` session need to restart.
  Documented three connection-check methods (`claude mcp list`,
  the VSCode "MCP servers" panel, and `/mcp` inside a session).

## [0.1.5] - 2026-05-02

### Fixed
- `src/main_src/start_cognee_mcp.py`: load `config/.env` into the
  process environment before spawning `cognee-mcp`. Previously the
  child process started without `LLM_API_KEY`, `LLM_ENDPOINT`,
  `SYSTEM_ROOT_DIRECTORY`, etc., causing MCP-side `cognify` calls
  to fail with `LLMAPIKeyNotSetError (Status 422)`. The fix uses a
  small stdlib-only loader so the script remains free of
  `python-dotenv` dependency from the system Python.

## [0.1.4] - 2026-05-02

### Added
- `docs/GETTING_STARTED.md`: added a "If sample ingestion fails"
  section after Step 1. Documents that `llama3.1:8b` may return
  unstable structured outputs that cause `InstructorRetryException`
  / `Field required` errors after 5 retries, and explains how to
  clean up with `delete_sample.py` and retry. Also suggests trying
  a larger model (e.g. `llama3.1:70b`) via `config/.env` `LLM_MODEL`
  if the failure persists.

## [0.1.3] - 2026-05-02

### Fixed
- `docs/SETUP.md` Section 4 ("Settings to update when relocating"):
  corrected `COGNEE_DATA_PATH` from **Required** to **Optional**.
  Step 2 already states that `COGNEE_DATA_PATH` can be left at its default
  value (`./data/cognee`), so the previous Section 4 entry contradicted
  Step 2 and confused users into thinking three paths must be edited when
  in fact only two (`SYSTEM_ROOT_DIRECTORY`, `DATA_ROOT_DIRECTORY`) are
  required.

## [0.1.2] - 2026-04-29

### Fixed
- `docs/SETUP.md` Step 5: the MCP registration command now requires an
  **absolute path** to `start_cognee_mcp.py`. The previous relative-path
  example (`src/main_src/start_cognee_mcp.py`) only worked when `claude` was
  launched from the cloned directory, because `--scope user` registers the
  server globally and Claude Code resolves relative paths against the current
  working directory at launch time. The corrected step shows how to reuse the
  `pwd` output from Step 2 to construct the absolute path, and adds a
  `Why an absolute path is required` note explaining the failure mode.

## [0.1.1] - 2026-04-29

### Fixed
- `docs/SETUP.md` Step 2: corrected wording from "three values" to "two values"
  (only `SYSTEM_ROOT_DIRECTORY` and `DATA_ROOT_DIRECTORY` need to be edited).
- `docs/SETUP.md` Step 3: corrected the `pip install` command. The previous
  `pip install "cognee[mcp][fastembed]"` was rejected by pip 24.0+ because
  `cognee 1.0.3` does not provide an `mcp` extra, and `cognee-mcp` is a
  separate package on PyPI. The verified working command is:

  ```
  pip install cognee-mcp "cognee[fastembed]"
  ```

  This installs `cognee-mcp 0.5.4`, `cognee 1.0.3`, and `fastmcp 3.2.4`.

## [0.1.0] - 2026-04-27

### Added
- Initial release.
- Cognee + Ollama + FastEmbed graph memory module for Claude Code.
- MCP server startup script (`src/main_src/start_cognee_mcp.py`).
- Sample knowledge ingestion script (`src/sample_src/load_sample.py`) with
  4 bundled markdown files.
- User-knowledge ingestion pipeline: `split_knowledge.py` (H2 splitter) and
  `import_knowledge.py` (per-chunk ingestion with retry on `cognify` failure).
- Setup and getting-started guides under `docs/`.
- MIT License.
