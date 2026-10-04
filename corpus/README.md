# Corpus

Put documents here (`.md`, `.txt`, `.rst`, `.pdf` with a text layer, plus common source suffixes).

On startup, if the index is empty, the service indexes this folder with both `structural` and `fixed` strategies. You can also trigger a rebuild:

```bash
curl -sS -X POST http://127.0.0.1:18766/v1/index \
  -H 'Content-Type: application/json' \
  -d '{"strategy":"structural"}'
```

`sample.md` is a tiny demo file so a fresh clone can search something immediately with `EMBEDDING_PROVIDER=fake`.
