# Architecture

```text
corpus/ + uploads
        │
        ▼
   extract_text ──► chunk (fixed | structural)
        │
        ▼
   embedder (api | local | fake)
        │
        ▼
   VectorStore ── SQLite chunks + vectors.npy + index_meta.json
        │
        ▼
   search: cosine top-k
        │
        ├── mode=raw       → return hits
        ├── mode=filtered  → drop score < min_score
        └── mode=full      → rewrite query → filter → heuristic rerank
```

## Components

| Module | Role |
|---|---|
| `settings.py` | Env via pydantic-settings |
| `chunking.py` | Fixed window + structural (Markdown headings) |
| `embeddings.py` | API / local / fake embedders |
| `store.py` | SQLite metadata + numpy matrix on disk |
| `rerank.py` | Query rewrite, score filter, token-overlap heuristic |
| `pipeline.py` | Index, heal, upload, search orchestration |
| `http_app.py` | FastAPI routes |
| `mcp_server.py` | Streamable HTTP MCP tools |
| `auth.py` | Optional Bearer middleware |

## Persistence

Under `RAG_DATA_DIR` (default `./data` in `.env.example`):

| File | Contents |
|---|---|
| `chunks.sqlite` | Chunk text + metadata (`scope`, `owner_id`, strategy, …) |
| `vectors.npy` | L2-normalized float32 matrix aligned with chunk ids |
| `index_meta.json` | Strategy / counts / last rebuild info |

Scopes:

- `stand` — corpus files from `RAG_CORPUS_DIR`
- `session` (default for uploads) — user docs, filterable by `owner_id`

## Startup heal

On boot the service starts a **background** heal task (does not block `/health`):

1. Empty index → reindex corpus with `structural` then `fixed`
2. Chunks without vectors → `ensure_vectors()` (API hang fuse → may fall back to fake)

Budgets live in `pipeline.py` (`REBUILD_BUDGET_S`, embed batch timeout).

## Retrieval modes

| Mode | Behavior |
|---|---|
| `raw` | Cosine top-k as stored |
| `filtered` | Drop hits below `RAG_MIN_SCORE` / request `min_score` |
| `full` | Lightweight query rewrite + filter + heuristic rerank (token overlap with title/section/text) |

`/v1/ask` uses the same search path and formats hits into `context` + Russian `prompt_suffix` for the caller’s LLM.

## Embeddings

| Provider | When |
|---|---|
| `fake` | Offline demos / CI; deterministic dims from `EMBEDDING_DIMS` |
| `api` | OpenAI-compatible (`LLM_BASE_URL` + key). Missing key → fake |
| `local` | `sentence-transformers` (`uv sync --extra local`) and `LOCAL_EMBEDDINGS_ENABLED=true` |

Flipping provider at runtime: `PATCH /v1/settings`. After a provider change, rebuild or heal so vector dims stay consistent.
