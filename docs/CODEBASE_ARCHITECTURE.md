# QMD Codebase Architecture Map

This document reverse engineers the repository structure from `src/` and `test/` to provide a software-architecture view of the code.

## High-level module map (`src/`)

- **`qmd.ts`** — CLI entry point and command router (`search`, `vsearch`, `query`, `get`, `multi-get`, `collection`, `context`, `mcp`, `status`, `update`, `embed`, `cleanup`).
- **`store.ts`** — Core data layer: indexing, chunking, BM25 search, vector search, hybrid search orchestration helpers, and document retrieval.
- **`db.ts`** — Database adapter for SQLite and `sqlite-vec` loading.
- **`llm.ts`** — LLM runtime wrapper (embedding, query expansion/generation, reranking, model lifecycle).
- **`mcp.ts`** — MCP server implementation for stdio/HTTP transports and MCP tool/resource registration.
- **`collections.ts`** — Collection and context config (`~/.config/qmd/*.yml`) management.
- **`formatter.ts`** — Output formatting for CLI formats (`default`, `json`, `files`, `csv`, `md`, `xml`).

## Runtime architecture

1. **CLI bootstrap** (`src/qmd.ts`)
   - Parses args, resolves index/config selection, initializes store, dispatches command handlers.
2. **Storage and retrieval** (`src/store.ts`)
   - Centralized access to document/content tables, embedding vectors, and collection-aware path resolution.
3. **Optional AI layer** (`src/llm.ts`)
   - Used by `query`, `vsearch`, and embedding workflows; not required for pure BM25 `search`.
4. **Protocol layer** (`src/mcp.ts`)
   - Exposes QMD functionality as MCP tools and `qmd://` resources.

## Core command flows

### `qmd search <query>`
- `qmd.ts` → `searchFTS(...)` in `store.ts` → formatter output.

### `qmd vsearch <query>`
- `qmd.ts` → embed query via `llm.ts` → `vectorSearchQuery(...)` in `store.ts` → formatter output.

### `qmd query <query>`
- `qmd.ts` → structured/hybrid query planning → lexical + vector sub-searches in `store.ts` → optional rerank via `llm.ts` → fused/ranked results.

### `qmd get` / `qmd multi-get`
- `qmd.ts` → path/docid normalization → retrieval helpers in `store.ts` → optional line/window formatting.

### `qmd mcp` (stdio or HTTP)
- `qmd.ts` starts MCP transport → `mcp.ts` registers tools/resources backed by `store.ts` + `llm.ts`.

## Data and config boundaries

- **Index DB**: `~/.cache/qmd/*.sqlite` (or `INDEX_PATH` override).
- **Collection config**: `~/.config/qmd/*.yml`.
- **Models/cache**: managed via `llm.ts` defaults under `~/.cache/qmd/models`.

## Test architecture (`test/`)

- **CLI integration**: `test/cli.test.ts`
- **MCP behavior**: `test/mcp.test.ts`
- **Store/search logic**: `test/store.test.ts`, `test/structured-search.test.ts`, `test/store.helpers.unit.test.ts`, `test/store-paths.test.ts`
- **Formatting**: `test/formatter.test.ts`
- **Collections config**: `test/collections-config.test.ts`
- **Multi-collection filtering**: `test/multi-collection-filter.test.ts`
- **LLM behavior (where applicable)**: `test/llm.test.ts`
- **Evaluation suites**: `test/eval*.test.ts`
