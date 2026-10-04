# Connect to AIChallenge Guest MCP

AIChallenge’s cloud API cannot open stdio MCP on your laptop. Publish this service as **Streamable HTTP** behind HTTPS, then paste URL + token into the site.

## Steps

1. **Configure token** in `.env`:

   ```bash
   RAG_SHARED_TOKEN=<long-random-string>
   RAG_HOST=127.0.0.1
   RAG_PORT=18766
   ```

2. **Start the service**

   ```bash
   uv sync
   uv run python -m aichallenge_rag
   ```

3. **Tunnel** (example with Cloudflare):

   ```bash
   cloudflared tunnel --url http://127.0.0.1:18766
   ```

   Copy the `https://….trycloudflare.com` URL.

4. **On AIChallenge** (logged in): Profile → **Подключения** / Guest MCP

   - **URL:** `https://<tunnel-host>/mcp`  
     (must include `/mcp`)
   - **Token:** exactly `RAG_SHARED_TOKEN`

5. **Smoke tools** from chat / MCP UI: `rag_stats` → `rag_search` with a query that matches your corpus.

## Stand compose (monorepo)

Inside [AIChallenge](https://github.com/ArtemKyslicyn/AIChallenge), the same code lives under `apps/rag` and is published on **loopback** `127.0.0.1:18766` by Docker Compose. Chat can use `use_rag` against the internal URL; external Guest MCP still needs a tunnel to that port.

## Checklist

- [ ] Token set and non-empty in production-like tunnels
- [ ] Corpus indexed (`rag_stats` shows chunks > 0)
- [ ] URL ends with `/mcp`
- [ ] Tunnel points at the RAG port, not the chat web UI
