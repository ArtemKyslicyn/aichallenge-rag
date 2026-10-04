# HTTP + MCP API

Base URL for local default: `http://127.0.0.1:18766`.

Auth header (when `RAG_SHARED_TOKEN` is set):

```http
Authorization: Bearer <RAG_SHARED_TOKEN>
```

Open routes without token: `/health`, `/`, `/docs`, `/openapi.json`.

## Health

```bash
curl -sS http://127.0.0.1:18766/health
# {"status":"ok"}
```

## Stats

```bash
curl -sS http://127.0.0.1:18766/v1/stats
```

Returns store counts plus `embedding_provider`, `chunk_strategy`, `embed_model_runtime`,
`vector_backend` (`qdrant`), `collection`, and optional `qdrant_url`.

## Index

```bash
curl -sS -X POST http://127.0.0.1:18766/v1/index \
  -H 'Content-Type: application/json' \
  -d '{"strategy":"structural"}'
```

`strategy`: `fixed` | `structural` (optional; defaults to settings).

## Search

```bash
curl -sS -X POST http://127.0.0.1:18766/v1/search \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "Guest MCP",
    "top_k": 6,
    "mode": "full",
    "owner_id": null
  }'
```

| Field | Notes |
|---|---|
| `query` | required |
| `top_k` | 1–32 |
| `mode` | `raw` \| `filtered` \| `full` |
| `top_k_pre` / `top_k_post` | candidate pool vs final keep |
| `min_score` | cosine floor for filtered/full |
| `owner_id` | limit to that owner’s uploads (+ stand, depending on pipeline rules) |

## Ask (context for an external LLM)

```bash
curl -sS -X POST http://127.0.0.1:18766/v1/ask \
  -H 'Content-Type: application/json' \
  -d '{"query":"chunk strategies","mode":"full"}'
```

Same retrieval as search, plus:

- `context` — numbered fragments
- `prompt_suffix` — instruction + context (Russian template)

## Upload document

```bash
curl -sS -X POST http://127.0.0.1:18766/v1/documents/upload \
  -F 'file=@./notes.md' \
  -F 'strategy=structural' \
  -F 'scope=session' \
  -F 'owner_id=demo-user'
```

Limits: ~5 MB body; PDF must have a text layer (not a scan). JSON text alternative: `POST /v1/documents`.

## List documents

```bash
curl -sS 'http://127.0.0.1:18766/v1/documents?owner_id=demo-user&include_stand=true'
```

## Settings

```bash
curl -sS -X PATCH http://127.0.0.1:18766/v1/settings \
  -H 'Content-Type: application/json' \
  -d '{"embedding_provider":"fake","chunk_strategy":"structural"}'
```

Rebuild the embedder in-process; reindex/heal if vector dims change.

## Heal

```bash
curl -sS -X POST http://127.0.0.1:18766/v1/heal
```

Use when chunks exist but `vectors.npy` is missing/empty after a failed API run.

## MCP (Streamable HTTP)

Mount: `/mcp`

Tools:

| Tool | Args | Purpose |
|---|---|---|
| `rag_stats` | — | Index + embed mode JSON |
| `rag_search` | `query`, `top_k=6`, `mode=full` | Search hits as JSON string |
| `rag_index` | `strategy=structural` | Reindex corpus |

Same Bearer as HTTP. For AIChallenge Guest MCP, see [connect-aichallenge.md](connect-aichallenge.md).
