# AIChallenge RAG sample

This file ships with the public repo so a local run can index and search without your own corpus.

## What this service does

`aichallenge-rag` builds a vector index over a folder of documents, then answers retrieval queries over HTTP and MCP.

## Chunk strategies

- **structural** — splits on Markdown headings and file boundaries.
- **fixed** — sliding window with overlap.

## Connect modes

- HTTP: `/v1/search`, `/v1/ask`, `/v1/documents/upload`
- MCP Streamable HTTP: `/mcp` tools `rag_stats`, `rag_search`, `rag_index`
