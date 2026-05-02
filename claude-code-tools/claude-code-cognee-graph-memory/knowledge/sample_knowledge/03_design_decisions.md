# Design decisions

## Why KuzuDB for the graph database

KuzuDB was chosen because it is Python-native, runs in-process, and is bundled
with Cognee (no extra installation). Compared with PostgreSQL or Neo4j, it was
the only option that satisfied the local-only and zero-cost requirements.

## Why MCP scope=user

The Cognee MCP server is registered with `scope=user`. With `scope=project`,
only that one project's Claude Code session could access it. Cross-project
access is the whole point of this graph memory system.

## Why FastEmbed all-MiniLM-L6-v2 for embeddings

FastEmbed's all-MiniLM-L6-v2 was chosen because it is bundled with Cognee, runs
fully locally without an external API key, and is lightweight (384 dimensions)
yet accurate. Versus OpenAI embeddings, it satisfies the zero-additional-cost
requirement.

## Why Ollama + llama3.1:8b for the LLM

Ollama with llama3.1:8b runs fully locally, needs no external API key, and
reuses an existing Ollama installation. It is accurate enough for entity
extraction and incurs no additional cost.

## Why stdio mode for transport

stdio mode is used to communicate with Claude Code because it does not consume
a port (no clashes with other services), and Claude Code launches the MCP
server directly as a child process — no separate HTTP server required.
