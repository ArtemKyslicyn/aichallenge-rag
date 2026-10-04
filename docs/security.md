# Security

## Threat model (short)

This service is a **document index with search and upload**. Anyone with the public URL and Bearer token can:

- reindex the corpus
- search all indexed text
- upload new documents into the index
- change embedding / chunk settings

Treat URL + token like a password for that machine’s RAG data.

## Hardening defaults

| Control | Recommendation |
|---|---|
| Bind | `RAG_HOST=127.0.0.1` (loopback) |
| Token | Long random `RAG_SHARED_TOKEN`; never commit `.env` |
| Ports | Do not bind product traffic to public `:443` / `:8443` |
| Tunnel | Prefer ephemeral or authenticated tunnels; rotate token if leaked |
| Corpus | Mount read-only in Docker (`:ro`) when possible |
| Uploads | Size-capped; PDF text-layer only — still, untrusted uploads enter your index |

Empty `RAG_SHARED_TOKEN` means **open** (local/dev convenience). Never leave that on a reachable network.

## What is *not* protected

- Content of indexed documents (search returns text)
- Side-channel size of corpus via stats
- Your embedding API key if set in the process environment (keep it out of git and chat)

## Secrets hygiene

- Commit only `.env.example` with empty placeholders
- Discuss variable **names**, never values
- Rotate `RAG_SHARED_TOKEN` and embedding keys if they appear in logs or tickets
