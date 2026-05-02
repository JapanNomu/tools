# Claude Code + Cognee Graph Memory System

**Version**: 0.1.11

A module that adds graph-based memory to Claude Code. It accumulates work-related memory (rules, lessons learned, design decisions, incident records) across sessions, enabling retrieval in later sessions.

---

### [Claude Code × Cognee — Practical Know-How Accumulation Tool]

**RAG gives you the answer. Cognee gives you the whole story.**
Claude Code remembers why decisions were made.
**Never say the same thing twice again.**

With the auto-accumulation harness in `harness/`, the more you use Claude Code, the more your own personal know-how is recorded into Cognee graph memory — and decisions, context, and related facts come back to you in a connected chain. See `docs/HARNESS_GUIDE.md` for details.

---

## Why Cognee MCP alone is not sufficient

1. **Cognee MCP only accepts file paths or text strings** — it has no built-in mechanism to automatically ingest Claude Code work logs or conversation text into graph memory
2. **`import_to_graph.py` bridges this gap** — it can ingest files from `~/.claude/rules/` or any directory into Cognee
3. **`start_cognee_mcp.py` connects Claude Code to Cognee** — without registering as an MCP server, Cognee cannot be used as a Claude Code tool

---

## Features

- **Fully local** — Ollama + FastEmbed. No external API keys required, zero additional cost
- **Cross-session** — Same graph memory accessible from any Claude Code session
- **Graph + vector search** — High-precision recall powered by KuzuDB (graph) + LanceDB (vector)
- **Role-separated folders** — Production runtime, sample handling, and user knowledge ingestion are separated by folder

---

## Reading order

| Order | Document | Content |
|---|---|---|
| 1 | This `README.md` | Overview, prerequisites, directory structure |
| 2 | `docs/SETUP.md` | Environment setup (venv creation, Ollama, MCP registration) |
| 3 | `docs/GETTING_STARTED.md` | Operation verification, usage, ingestion of your own knowledge |
| 4 | `docs/HARNESS_GUIDE.md` | Auto-accumulation harness setup (optional, strongly recommended) |

---

## Prerequisites

### Verified environment

| Item | Value |
|------|---|
| OS | Linux (Ubuntu 22.04 or later) / WSL2 |
| Python | 3.12 or higher |
| Ollama | Latest version (when using local LLM) |
| LLM | qwen2.5:14b (num_ctx=8192) — local default. Cloud APIs (Claude / OpenAI) are strongly recommended for production. |
| Claude Code | Latest version |

### Recommended hardware

| Use mode | GPU | RAM | LLM |
|---------|-----|-----|-----|
| **Cloud API (strongly recommended)** | Not required | 16GB+ | claude-sonnet-4-6 / gpt-4o, etc. |
| Local LLM (recommended) | RTX 4070 12GB+ | 32GB+ | qwen2.5:32b or larger |
| Local LLM (verified minimum) | RTX 4060 8GB | 32GB | qwen2.5:14b — works but response time is noticeably slower |

See `docs/GETTING_STARTED.md` "Recommended LLM and Environment" for details.

### Technology stack

| Technology | Details |
|------|------|
| Graph memory engine | Cognee |
| LLM (entity extraction) | qwen2.5:14b (default, local) / Claude API / OpenAI API |
| LLM runtime | Ollama (local) or cloud API |
| Graph DB | KuzuDB (bundled with Cognee) |
| Vector DB | LanceDB (bundled with Cognee) |
| Embedding model | FastEmbed all-MiniLM-L6-v2 |

---

## Directory structure

```
distribution root/
├── README.md                     ← this file
├── LICENSE                       MIT License
│
├── config/
│   └── .env.example              Environment variable template (copy to config/.env)
│
├── docs/
│   ├── SETUP.md                  Environment setup guide
│   ├── GETTING_STARTED.md        Operation verification & usage
│   └── HARNESS_GUIDE.md          Auto-accumulation harness setup
│
├── harness/                      Claude Code × Cognee auto-accumulation harness (optional, strongly recommended)
│   ├── CLAUDE_md_sample.md       Snippet to append to your project's CLAUDE.md
│   ├── rules/
│   │   └── cognee_memory_usage.md   Long-form rule for ~/.claude/rules/
│   ├── hooks/
│   │   ├── auto_remember_user_message.py    UserPromptSubmit hook
│   │   ├── auto_remember_completion.py      Stop hook
│   │   └── cognee_remember_flusher.py       Queue drainer (cron recommended)
│   └── settings.example.json     Merge into ~/.claude/settings.json
│
├── src/
│   ├── main_src/                 Production runtime (active during Claude Code sessions)
│   │   ├── start_cognee_mcp.py   MCP server startup script
│   │   └── import_to_graph.py    Production ingestion (called from Claude Code)
│   ├── sample_src/               Sample-related operations
│   │   ├── load_sample.py        Load bundled samples
│   │   └── delete_sample.py      Delete bundled samples
│   └── knowledge_src/            Initial ingestion of your own knowledge (mitigates cognify failures, supports batched execution)
│       ├── split_knowledge.py    Split files (by H2 heading)
│       └── import_knowledge.py   Ingest with retry support
│
└── knowledge/
    ├── sample_knowledge/         Bundled sample data (4 files)
    ├── user_knowledge/           Place your knowledge source files here (see folder README)
    └── user_chunks/              Auto-generated split files for ingestion
```

---

## Quick start

1. Follow `docs/SETUP.md` to set up your environment
2. Run `src/venv/bin/python3 src/sample_src/load_sample.py` to load sample data
3. Try the example queries in `docs/GETTING_STARTED.md`
4. Follow `docs/HARNESS_GUIDE.md` to enable the harness (**strongly recommended** — this is what makes your know-how accumulate automatically)

To ingest your own knowledge, see "Step 4: Ingest your own knowledge" in `docs/GETTING_STARTED.md`.

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

Copyright (c) 2026 JapanNomu
