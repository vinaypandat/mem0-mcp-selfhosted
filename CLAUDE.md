# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## MCP Servers

- **mem0**: Persistent memory across sessions. At the start of each session, `search_memories` for relevant context before asking the user to re-explain anything. Use `add_memory` whenever you discover project architecture, coding conventions, debugging insights, key decisions, or user preferences. Use `update_memory` when prior context changes. When in doubt, save it — future sessions benefit from over-remembering.

## Build & Test Commands

```bash
pip install -e ".[dev]"              # Install with dev dependencies
python3 -m pytest tests/unit/ -v     # Unit tests (mocked, no infra needed)
python3 -m pytest tests/contract/ -v # Contract tests (validates mem0ai internals)
python3 -m pytest tests/integration/ -v  # Integration tests (requires live Qdrant + Neo4j + Ollama)
python3 -m pytest tests/ -m "not integration" -v  # Skip integration tests
python3 -m pytest tests/unit/test_auth.py::TestIsOatToken::test_oat_token_detected -v  # Single test
mem0-mcp-selfhosted                  # Launch MCP server (stdio transport by default)
mem0-install-hooks                   # Install Claude Code session hooks
```

## Architecture

Self-hosted MCP server using `mem0ai` as a library. 11 MCP tools (7 memory + 2 entity + 2 graph), FastMCP orchestrator. Memory is initialized lazily on first tool call (30s retry cooldown on failure) — server responds to `initialize` and `tools/list` without live infrastructure.

**Module roles:**
- `server.py` — FastMCP orchestrator. Registers all tools + `memory_assistant` prompt. Defers `Memory` init to first tool call via `_ensure_memory()` with thread-safe lock. After Memory creation, swaps in `SplitModelGraphLLM` router if `gemini_split` provider is configured.
- `config.py` — Env vars → mem0ai `MemoryConfig` dict. Returns `(config_dict, providers_info, split_config)`. Handles 5 graph LLM provider configs and URL/provider cascade chains.
- `auth.py` — 4-tier token fallback: `MEM0_ANTHROPIC_TOKEN` → `~/.claude/.credentials.json` → `ANTHROPIC_API_KEY` → `None`. Includes OAT token detection, expiry checking, and OAuth refresh logic.
- `llm_anthropic.py` — Custom Anthropic LLM provider (`AnthropicOATLLM`) registered with mem0ai's `LlmFactory`. Fixes upstream tool-call parsing, adds structured outputs via `output_config` (schema selection based on whether system message is present — see contract tests), handles OAT token auto-refresh with 3-step strategy (piggyback → self-refresh → wait-and-retry), and proactive pre-expiry refresh.
- `llm_ollama.py` — Custom Ollama provider (`OllamaToolLLM`). Restores tool-calling removed in mem0ai upstream. Adds 6 defensive layers against `<think>`+`format:json` incompatibility (no_think injection, deterministic temps, think-tag stripping, JSON extraction, single retry). See Ollama issues #10538, #10929, #10976.
- `llm_router.py` — `SplitModelGraphLLM` routes graph pipeline calls by tool name: extraction tools (`extract_entities`, `establish_relationships`) → Gemini; contradiction tools (`add_graph_memory`, etc.) → Claude.
- `helpers.py` — `_mem0_call()` error wrapper, `call_with_graph()` threading lock for per-call graph toggle, `safe_bulk_delete()` iterates+deletes individually, `patch_graph_sanitizer()` and `patch_gemini_parse_response()` monkey-patches, entity listing via Qdrant Facet API with scroll fallback for older Qdrant.
- `graph_tools.py` — Direct Neo4j Cypher queries for `mcp_search_graph` and `mcp_get_entity` tools with lazy driver init.
- `hooks.py` — Session hooks: `context_main()` (SessionStart → injects memories as `additionalContext`) and `stop_main()` (Stop → saves session summary with `infer=True`, graph force-disabled for speed). Install via `mem0-install-hooks`.
- `env.py` — Centralized env var readers with whitespace stripping (guards against `.env` trailing newlines).
- `__init__.py` — Sets `MEM0_TELEMETRY=false` **before** any mem0 import (prevents PostHog). Defines `main()` entry point.

**Critical implementation details:**
- `memory.delete()` does NOT clean Neo4j nodes (mem0ai bug #3245) — `safe_bulk_delete()` explicitly calls `memory.graph.delete_all(filters)` after each delete. **Never call `memory.delete_all()`** (triggers `vector_store.reset()`).
- `memory.enable_graph` is mutable instance state — `call_with_graph()` holds a `threading.Lock` for the full duration of each Memory call (2-20s unavoidable due to `concurrent.futures.wait()` internally).
- Contract tests (`tests/contract/`) validate mem0ai internal API assumptions — if these fail after a mem0ai upgrade, the custom providers need updating before anything else.
- `Memory.update()` uses `data=` parameter, not `text=`.
- Structured output schema selection in `llm_anthropic.py` relies on a mem0ai invariant: fact extraction calls have a system message; memory update calls do not. `test_schema_detection.py` validates this.
- `MEM0_ENABLE_GRAPH=false` by default (graph adds 3 extra LLM calls per `add_memory`).
- `NEO4J_DATABASE` env var workaround: mem0ai passes config as positional args where 4th param is `token`, not `database` — set `NEO4J_DATABASE` instead (read via langchain_neo4j's `get_from_dict_or_env()`).
- Monkey-patches (`patch_graph_sanitizer`, `patch_gemini_parse_response`) must run after mem0 modules are imported but before `Memory.from_config()`.
- Structured outputs require claude-opus-4/sonnet-4/haiku-4 series; older models fall back to JSON extraction.
- `MEM0_TELEMETRY=false` must be set before any mem0 import — cannot be disabled dynamically.

**Key environment variables:**

| Variable | Default | Purpose |
|---|---|---|
| `MEM0_PROVIDER` | `anthropic` | Top-level cascade for LLM + graph provider |
| `MEM0_LLM_MODEL` | `claude-opus-4-6` | Main LLM model |
| `MEM0_ENABLE_GRAPH` | `false` | Enable Neo4j knowledge graph |
| `MEM0_GRAPH_LLM_PROVIDER` | cascades | `anthropic`, `ollama`, `gemini`, `gemini_split` |
| `MEM0_OLLAMA_URL` | `http://localhost:11434` | Cascades to LLM/embed/graph URLs |
| `MEM0_QDRANT_URL` | `http://localhost:6333` | Qdrant REST endpoint |
| `MEM0_NEO4J_URL` | `bolt://127.0.0.1:7687` | Neo4j bolt endpoint |
| `MEM0_TRANSPORT` | `stdio` | `stdio`, `sse`, or `streamable-http` |
| `MEM0_USER_ID` | `user` | Default user ID injected into all tool calls |
| `MEM0_ANTHROPIC_TOKEN` | — | Priority 1 token (OAT); falls back to `.credentials.json` then `ANTHROPIC_API_KEY` |
| `GOOGLE_API_KEY` | — | Required for Gemini graph LLM providers |
| `MEM0_OAT_REFRESH_THRESHOLD_SECONDS` | `1800` | Proactive token refresh threshold |
