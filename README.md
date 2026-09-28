# Codebase-MCP-Server

A **Model Context Protocol (MCP) server** that turns any codebase into a
queryable, citation-grounded knowledge base for your AI assistant. Point it at
a repo — it parses, chunks, indexes, and serves six tools over stdio that
Claude Desktop, Claude Code, Cursor, or any MCP client can call.

## What it does

```
 repo ──▶ ingest ──▶ parse ──▶ chunk ──▶ index ──▶ serve (MCP tools)
          (git/     (tree-     (AST-     (FAISS +
           local)    sitter)    aware)    BM25)
```

| MCP tool | Purpose |
|---|---|
| `search_code` | Hybrid vector + BM25 search with cross-encoder reranking |
| `find_symbol` | Jump to a function/class definition |
| `get_file_context` | Read a file slice around a symbol |
| `ask` | Natural-language Q&A with citations, multi-provider LLM |
| `analyze_impact` | "What breaks if I change this symbol?" impact analysis |
| `repository_summary` | High-level map of the repo |

**Pipeline highlights:**

- **Tree-sitter AST-aware chunking** — chunks follow symbol boundaries
  (functions/classes), with a line-based fallback for oversized symbols.
- **Hybrid retrieval** — dense FAISS vectors + sparse BM25 fused with
  Reciprocal Rank Fusion, then reordered by a local cross-encoder reranker.
- **Incremental indexing** — per-file cache: only changed files are
  re-parsed and re-embedded on re-index.
- **Swappable LLM providers** — NVIDIA, Groq, OpenRouter, Gemini, or any local
  OpenAI-compatible endpoint (Ollama/vLLM/LM Studio); everything via env vars.
- **Zero-config serving** — `serve` auto-indexes the repo in the background on
  first connect instead of failing; tool calls report "still building" until
  the index is ready.

## Quickstart

```bash
pip install -e ".[dev]"
cp .env.example .env   # add at least one LLM provider key for `ask`

# Build the index for a repo
codebase-rag index /path/to/repo --index-dir /tmp/my-index

# Serve over MCP stdio (Claude Desktop / Claude Code / Cursor)
codebase-rag serve --repo /path/to/repo --index-dir /tmp/my-index
```

Claude Desktop config snippet:

```json
{
  "mcpServers": {
    "codebase-rag": {
      "command": "codebase-rag",
      "args": ["serve", "--repo", "/path/to/repo"],
      "env": { "GROQ_API_KEY": "..." }
    }
  }
}
```

## Configuration

All settings are environment variables (see `.env.example`) — provider keys,
embedding/reranker model names, retrieval weights, log level, index location.
A `.env` file is loaded for local development; MCP clients should use their
own server-registration `env` block instead.

## Project structure

```
├── src/codebase_rag_mcp/
│   ├── cli/          # `codebase-rag index|serve` entry points
│   ├── mcp/          # MCP server: 6 tools over stdio
│   ├── ingestion/    # git/local repo loading, language filters
│   ├── parser/       # tree-sitter symbol extraction
│   ├── chunker/      # AST-aware chunking + oversized-symbol fallback
│   ├── indexing/     # FAISS vector, BM25, manifest, incremental cache
│   ├── retrieval/    # hybrid search (RRF fusion)
│   ├── reranker/     # cross-encoder reranking
│   ├── generation/   # multi-provider LLM answers with citations
│   ├── impact/       # change-impact analysis
│   └── citations/    # deterministic citation attach/format
├── tests/            # pytest suite
├── benchmarks/       # retrieval benchmark harness
└── .github/workflows # ruff + mypy + pytest CI
```

## Notes

- Embeddings and reranking run locally via `sentence-transformers`
  (`all-MiniLM-L6-v2` / `cross-encoder/ms-marco-MiniLM-L-6-v2` by default).
- Tested with Python 3.11+.

## License

MIT — see [LICENSE](LICENSE).
