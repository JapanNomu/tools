# GETTING STARTED — Claude Code + Cognee Graph Memory: Operation Verification & Usage

This document explains how to verify the system after setup and how to ingest your own knowledge.

If you have not completed setup yet, please refer to `docs/SETUP.md` first.

---

## Step 1: Verify operation with bundled samples

Run the following in your terminal.

```bash
cd <cloned directory>
src/venv/bin/python3 src/sample_src/load_sample.py
```

The four files under `knowledge/sample_knowledge/` will be ingested into Cognee one by one.

```
2026-04-29 10:00:00 [INFO] [1/4] 01_claude_code_tips.md → dataset=sample_knowledge
2026-04-29 10:00:30 [INFO] [2/4] 02_software_dev_lessons.md → dataset=sample_knowledge
...
```

Time estimate: 2-5 minutes (includes Ollama graph processing).

### If sample ingestion fails

The LLM (`llama3.1:8b`) sometimes returns unstable responses that cause structured-output validation errors. After 5 retries the script may abort with errors such as `InstructorRetryException` or `Field required`. If this happens, clean up and retry:

```bash
# Remove any partially-ingested sample data
src/venv/bin/python3 src/sample_src/delete_sample.py

# Re-ingest the samples
src/venv/bin/python3 src/sample_src/load_sample.py
```

If the failure persists, check that Ollama is reachable (`ollama list` should show the model), or try a larger model (e.g. `llama3.1:70b`) by changing `LLM_MODEL` in `config/.env`.

---

## Step 2: Launch Claude Code and call MCP tools

Open a new Claude Code session (it can be a different session from the ingestion).

Try the following queries in the chat to retrieve knowledge from graph memory.

---

### Scenario A: Ask about Claude Code usage

**Your input:**

```
search("Tell me about the timing of git push", search_type="CHUNKS")
```

Or in natural language:

```
recall("When can I run git push?")
```

> ⚠️ With llama3.1:8b, `recall` may fail with an "LLM format error". If it fails, use `search(query, search_type="CHUNKS")` as a fallback (see Troubleshooting at the bottom of this file).

**Expected response:**

> "Run `git push` only when the user explicitly instructs to. Task or phase completion is not a reason to push."

---

### Scenario B: Look up past errors

```
search("How to handle errors when Ollama is unreachable", search_type="CHUNKS")
```

**Expected response:**

> "Run `ollama serve` and retry. Also check whether `llama3.1:8b` is downloaded with `ollama list`."

---

### Scenario C: Retrieve design rationale

```
search("Why KuzuDB is used", search_type="CHUNKS")
```

**Expected response:**

> "Python-native, in-process execution, bundled with Cognee (no extra installation). It was the only choice that satisfied the local-only and zero-cost requirements."

---

### Scenario D: Retrieve development lessons

```
search("Where to derive test expected values from", search_type="CHUNKS")
```

**Expected response:**

> "Unit-test expected values must be derived from the IF specification. Reading the implementation code to determine expected values is forbidden."

---

## Step 3: Register your own knowledge ad hoc (single record)

You can register knowledge gained on the fly by calling the `remember` tool from Claude Code.

```
remember("Today's lesson: Always take a backup before running Django migrate. There was a non-rollback-capable table change.", dataset_name="my_lessons")
```

In a later session:

```
recall("What should I watch out for with Django migrate?")
```

Returns the knowledge you registered.

> ⚠️ If `recall` fails with an "LLM format error", use `search("Django migrate", search_type="CHUNKS")` as a fallback.

---

## Step 4: Ingest your own knowledge in bulk

Steps to ingest existing knowledge files (`.md`).

### Step 4-1: Delete sample data (optional)

If you no longer need the bundled samples:

```bash
src/venv/bin/python3 src/sample_src/delete_sample.py
```

Only the `sample_knowledge` dataset is removed.

### Step 4-2: Place your knowledge under user_knowledge/

Place `.md` files under `knowledge/user_knowledge/`. You can use sub-folders by category.

```
knowledge/user_knowledge/
├── project-management/
│   └── task-management.md
├── design/
│   └── db-design.md
└── lessons/
    └── past-incidents.md
```

### Step 4-3: Split the knowledge

```bash
src/venv/bin/python3 src/knowledge_src/split_knowledge.py
```

Each `.md` under `user_knowledge/` is split by H2 heading and written to `knowledge/user_chunks/`.

### Step 4-4: Ingest the chunks

```bash
src/venv/bin/python3 src/knowledge_src/import_knowledge.py
```

Files in `user_chunks/` are ingested into Cognee one at a time, with retries on cognify failure.

Time estimate: tens of seconds to a few minutes per file. For 100 files, roughly tens of minutes to several hours.

### Step 4-5: Preview the ingestion list

```bash
src/venv/bin/python3 src/knowledge_src/import_knowledge.py --dry-run
```

A dry-run shows the file list that would be ingested.

---

## Tool reference

| Tool | Purpose | Example |
|-------|------|---|
| `remember(data, dataset_name)` | Register knowledge / decisions / lessons | `remember("...", dataset_name="lessons")` |
| `recall(query)` | Semantic search using graph + LLM (fallback to `search` on failure) | `recall("What were past incidents?")` |
| `search(search_query, search_type="CHUNKS")` | Retrieve text directly via vector search | `search("How to handle errors", search_type="CHUNKS")` |
| `list_data()` | Show registered datasets | `list_data()` |
| `prune()` | Reset all data | `prune()` |

---

## Troubleshooting

**SearchPreconditionError**
→ No data has been ingested yet. Run Step 1 first.

**`recall` returns empty results**
→ Graph processing might still be running. Check completion with `cognify_status()` and retry.

**LLM format error during `recall`**
→ llama3.1:8b sometimes does not respond in the JSON format Cognee expects. Use `search(query, search_type="CHUNKS")` as a fallback.

**Knowledge ingestion shows `status=errored`**
→ The file may be too large. Run `split_knowledge.py` to split it before ingesting. `import_knowledge.py` retries up to 3 times on failure.
