# aichallenge-rag

**Standalone RAG index / search service** for [AIChallenge](https://github.com/ArtemKyslicyn/AIChallenge) and any client that can call HTTP or Streamable HTTP MCP.

Index a folder of docs (or upload files), retrieve chunks with scores, optionally filter + heuristic-rerank, then feed the context into *your* LLM. This process never calls a chat model for answers — only embeddings (API, local, or fake).

| You run | AIChallenge / client sees |
|---|---|
| Index + search on your machine (or stand) | HTTP `/v1/*` or Guest MCP `/mcp` |
| SQLite + numpy vectors under `RAG_DATA_DIR` | Bearer token only |

Origin in the monorepo: `apps/rag`. This repo is the public, self-contained copy.

## Features

- **Corpus index** — walk `RAG_CORPUS_DIR` (md/txt/pdf with text layer, common source suffixes)
- **Two chunk strategies** — `fixed` window vs `structural` (headings / files)
- **User uploads** — `POST /v1/documents/upload` with optional `owner_id` scope
- **Retrieval modes** — `raw` · `filtered` (min score) · `full` (rewrite + filter + heuristic rerank)
- **`/v1/ask`** — returns a ready `context` / `prompt_suffix` block (no LLM inside)
- **MCP tools** — `rag_stats`, `rag_search`, `rag_index` on `/mcp`
- **Embeddings** — OpenAI-compatible API, optional `sentence-transformers`, or deterministic `fake` for offline demos

## 5-minute local run

```bash
git clone https://github.com/ArtemKyslicyn/aichallenge-rag.git
cd aichallenge-rag
cp .env.example .env
# optional: set RAG_SHARED_TOKEN to a long random string
uv sync
uv run python -m aichallenge_rag
```

- Health: [http://127.0.0.1:18766/health](http://127.0.0.1:18766/health)
- OpenAPI: [http://127.0.0.1:18766/docs](http://127.0.0.1:18766/docs)

With the bundled `corpus/sample.md` and `EMBEDDING_PROVIDER=fake`:

```bash
curl -sS -X POST http://127.0.0.1:18766/v1/search \
  -H 'Content-Type: application/json' \
  -d '{"query":"chunk strategies","mode":"full"}'
```

Tests:

```bash
uv run pytest
```

## Connect to AIChallenge (Guest MCP)

1. Run the service on loopback (`RAG_HOST=127.0.0.1`).
2. Set the same long token in `.env` as `RAG_SHARED_TOKEN`.
3. Tunnel HTTPS to the process, e.g. `cloudflared tunnel --url http://127.0.0.1:18766`.
4. On the site: **Profile → Подключения (Guest MCP)**  
   - URL: `https://<tunnel-host>/mcp`  
   - Token: same as `RAG_SHARED_TOKEN`

Full walkthrough: [docs/connect-aichallenge.md](docs/connect-aichallenge.md).

## HTTP API (summary)

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/health` | Liveness (no auth) |
| `GET` | `/v1/stats` | Chunk / vector / embed mode |
| `POST` | `/v1/index` | Rebuild corpus (`fixed` \| `structural`) |
| `POST` | `/v1/heal` | Rebuild vectors if chunks exist without matrix |
| `POST` | `/v1/documents` | Add plain-text document (JSON) |
| `POST` | `/v1/documents/upload` | Upload file (multipart) |
| `GET` | `/v1/documents` | List docs (`owner_id`, `include_stand`) |
| `POST` | `/v1/search` | Retrieve hits + scores |
| `POST` | `/v1/ask` | Search + `context` / `prompt_suffix` |
| `PATCH` | `/v1/settings` | Flip embed provider / chunk strategy |

Details and examples: [docs/api.md](docs/api.md).

When `RAG_SHARED_TOKEN` is set, send `Authorization: Bearer <token>` on all routes except `/health`, `/`, `/docs`, `/openapi.json`.

## Docker

```bash
docker build -t aichallenge-rag .
docker run --rm -p 127.0.0.1:18766:18766 \
  -e EMBEDDING_PROVIDER=fake \
  -e RAG_CORPUS_DIR=/app/corpus \
  -e RAG_DATA_DIR=/app/data \
  -v "$PWD/corpus:/app/corpus:ro" \
  -v "$PWD/data:/app/data" \
  aichallenge-rag
```

Or: `docker compose up --build` (binds loopback only — see `docker-compose.yml`).

Never publish this service on public `:443` / `:8443` without your own edge plan. Prefer loopback + tunnel.

Optional local embeddings image:

```bash
uv sync --extra local
# or bake sentence-transformers into a custom image, then:
# PATCH /v1/settings {"local_embeddings": true}
```

## Docs

| Doc | Topic |
|---|---|
| [docs/architecture.md](docs/architecture.md) | Pipeline, store, modes |
| [docs/api.md](docs/api.md) | HTTP + MCP contracts |
| [docs/connect-aichallenge.md](docs/connect-aichallenge.md) | Tunnel + Guest MCP |
| [docs/security.md](docs/security.md) | Tokens, bind, risks |

## Env (names only)

See [`.env.example`](.env.example). Important names:

- `RAG_SHARED_TOKEN` — Bearer for HTTP + MCP (empty = open local/dev)
- `RAG_CORPUS_DIR` / `RAG_DATA_DIR` — corpus + SQLite/numpy index
- `EMBEDDING_PROVIDER` — `api` \| `local` \| `fake`
- `LLM_API_KEY` / `ROUTERAI_KEY` — for API embeddings
- `RAG_MODE` / `RAG_MIN_SCORE` / `RAG_TOP_K_*` — retrieval defaults

## Requirements

- Python 3.12+
- [uv](https://docs.astral.sh/uv/) recommended
- Optional: embedding API key, or `uv sync --extra local` for sentence-transformers
- Optional: [cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation/) / ngrok for Guest MCP

## Security

Whoever has **URL + Bearer** can reindex, search, and upload into *your* index. Keep bind on loopback; only expose via a tunnel you control. See [docs/security.md](docs/security.md).

## License

MIT — see [LICENSE](LICENSE).
