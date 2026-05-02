# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
