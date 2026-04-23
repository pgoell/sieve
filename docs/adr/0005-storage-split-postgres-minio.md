# ADR 0005: Storage split: Postgres + pgvector for data and vectors, MinIO for blobs

**Status:** Proposed
**Date:** 2026-04-23

## Context

Sieve stores three kinds of data with very different characteristics:

1. **Relational** (small rows, heavy read/write, transactional): users, sources, subscriptions, item metadata, digest definitions, read state.
2. **Vector** (fixed-size embeddings, k-NN queries): one vector per item for semantic search.
3. **Blob** (large, mostly-append, rarely-mutated, retrieved by key): full HTML content, raw MIME emails, later audio/video.

Options considered:

1. **Everything in Postgres** (TEXT/BYTEA for blobs). Simplest, but Postgres backups explode and query performance on large-blob tables suffers.
2. **Everything in a single analytical/blob store** (DuckDB, Databricks, BigQuery + blob store). DuckDB is analytical, not OLTP; Databricks is orders of magnitude too expensive and too heavy.
3. **Dedicated vector DB** (Qdrant, Weaviate, Pinecone) + Postgres + object store. More moving parts; unnecessary at Phase 1 scale.
4. **Postgres (+pgvector) for relational + vectors; object store for blobs.** Two stores, clean separation of concerns.

## Decision

Use option 4.

- **Postgres (PostgreSQL 17 with the `vector` extension).** Holds all relational data, the `items.embedding` column (pgvector), and the `jobs` table. One backup, one restore path. HNSW indexes for k-NN queries.
- **MinIO.** Self-hosted S3-compatible store on the same VPS. Keys of the form `content/<source_id>/<item_id>.html` and `mime/<user_id>/<message_id>.eml`. Buckets: `sieve-content`, `sieve-mime`. Versioning off in Phase 1; retention policy configurable later.

### Blob access pattern

- Application writes to MinIO via the S3 protocol using app-scoped credentials.
- `items.content_blob_key` stores the key; the app fetches by key at read time.
- The blob is never inlined into API responses; it is fetched and sanitized on demand for the reader view.

### Vector access pattern

- Embeddings generated per item (dims depend on provider, commonly 1536 or 3072).
- Column `items.embedding vector(dims)` with an HNSW index built lazily.
- Phase 2 semantic search: `embed(query) → SELECT items ORDER BY embedding <=> :query_vec LIMIT k`, joined through the subscription gate.

### Growth path

- MinIO can move to Hetzner Storage Box or Cloudflare R2 by changing endpoint + credentials in env. Keys and bucket names stay the same.
- pgvector handles millions of rows at this dim count comfortably; upgrading to a dedicated vector DB is a Phase 4+ concern, if ever.

## Consequences

### Easier

- One database to operate for OLTP + vectors. One set of transactions. `items` row and its embedding can be inserted in one statement.
- Object storage is on a standard, widely-supported API (S3); migrations between backends are trivial.
- Backup sizes are bounded; Postgres dumps are small, MinIO can be backed up or replicated independently.
- No extra stateful service in Phase 1 beyond Postgres and MinIO.

### Harder

- Reader view requires a round trip to MinIO on item expand. Acceptable latency; MinIO is on the same host. Cacheable at the application layer if ever needed.
- Object store and database can drift (blob deleted, key still referenced) under a crash. Mitigated by: blobs are written first, then the row with the key; delete flow marks row deleted, then deletes blob.

### Rejected alternatives

- **Postgres for blobs (BYTEA/TEXT):** bloats tables, slows queries, inflates backup size, and locks us into Postgres-scale blob economics.
- **Databricks / BigQuery:** cloud-first, analytical, expensive; wrong shape for a trust-group OLTP app.
- **DuckDB:** excellent for analytical queries over an exported archive; not an OLTP database. Candidate for future offline analytics, not primary storage.
- **Dedicated vector DB (Qdrant, Weaviate, Pinecone):** another stateful service with its own backup and restore story. Only worth it at scales Sieve will not reach in the planned phases.
