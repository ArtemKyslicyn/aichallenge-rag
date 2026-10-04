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
   Qdrant collection ── dense COSINE vectors + payload
        │
        ▼
   query_points (ANN) + metadata filter
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
| `store.py` | **Qdrant** client (embedded path or URL) |
| `rerank.py` | Query rewrite, score filter, token-overlap heuristic |
| `pipeline.py` | Index, heal, upload, search orchestration |
| `http_app.py` | FastAPI routes |
| `mcp_server.py` | Streamable HTTP MCP tools |
| `auth.py` | Optional Bearer middleware |

## Qdrant persistence

| Mode | Location |
|---|---|
| Embedded | `RAG_DATA_DIR/qdrant/` + `index_meta.json` |
| Remote | `QDRANT_URL` collection `QDRANT_COLLECTION` |

Each point:

- **id** — UUID5 derived from `chunk_id`
- **vector** — L2-normalized dense embedding
- **payload** — `chunk_id`, `text`, `source`, `title`, `section`, `strategy`, `index`, `scope`, `owner_id`, `embed_model`

Visibility filter on search:

- `scope=stand` → always visible
- `scope=session` → only when `owner_id` matches the request

## Chunk scopes

- `stand` — corpus files from `RAG_CORPUS_DIR`
- `session` (default for uploads) — user docs, filterable by `owner_id`

## Startup heal

On boot the service starts a **background** heal task (does not block `/health`):

1. Empty collection → reindex corpus with `structural` then `fixed`
2. Otherwise leave the existing Qdrant points alone

## Retrieval modes

| Mode | Behavior |
|---|---|
| `raw` | Qdrant top-k as returned |
| `filtered` | Drop hits below `RAG_MIN_SCORE` / request `min_score` |
| `full` | Lightweight query rewrite + filter + heuristic rerank |

`/v1/ask` uses the same search path and formats hits into `context` + Russian `prompt_suffix` for the caller’s LLM.

## Embeddings

| Provider | When |
|---|---|
| `fake` | Offline demos / CI; deterministic dims from `EMBEDDING_DIMS` |
| `api` | OpenAI-compatible (`LLM_BASE_URL` + key). Missing key → fake |
| `local` | `sentence-transformers` (`uv sync --extra local`) and `LOCAL_EMBEDDINGS_ENABLED=true` |

Changing embed dims/model recreates the collection when incompatible.
