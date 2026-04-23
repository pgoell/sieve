# Architecture Overview

## One-paragraph summary

Sieve is a FastAPI web container plus a Python worker container, sharing a Postgres instance (with pgvector for embeddings and `SELECT ... FOR UPDATE SKIP LOCKED` for the job queue) and a MinIO object store for content blobs. Inbound email arrives via a Cloudflare Email Worker that POSTs raw RFC822 to an authenticated webhook on the web container. The React SPA is served as static assets by the web container behind Caddy. LLM calls use Anthropic or Google Gemini, resolved per-user (BYOK default, platform key for paid users).

## System diagram

```
          ┌──────────┐       ┌────────────────────┐
Browser ──▶  Caddy   ├──────▶│  FastAPI (web)     │
          └──────────┘       │  REST API + auth   │
             HTTPS           │  renders digests   │
                             │  exposes agent API │
                             └─────────┬──────────┘
                                       │
                                       ▼
                          ┌─────────────────────────┐
                          │  Postgres + pgvector    │◀──┐
                          │  relational + vectors + │   │
                          │  DB-backed job queue    │   │
                          └───────────┬─────────────┘   │
                                      │                 │
                                      ▼                 │
                           ┌─────────────────────┐      │
                           │  Worker container   │──────┘
                           │  APScheduler        │
                           │  ingestion          │
                           │  summarization      │
                           │  embedding          │
                           │  digest building    │
                           └───────────┬─────────┘
                                       │
                                       ▼
                             ┌──────────────────┐
                             │      MinIO       │
                             │  full HTML,      │
                             │  raw MIME,       │
                             │  (later) audio   │
                             └──────────────────┘

                      ┌─────────────────────────────────┐
Inbound email ───────▶│ Cloudflare Email Worker         │
(*@in.digest…)        │ parses, POSTs RFC822 to web API │
                      └────────────┬────────────────────┘
                                   │ HTTPS webhook
                                   ▼
                    (FastAPI /webhooks/inbound-email)
```

## Components

### Web container

FastAPI. Serves:

- REST API for the SPA and (Phase 3) agents.
- Static React build assets.
- Webhook endpoints (`/webhooks/inbound-email`).
- Admin endpoints for source approval.

Stateless. Single replica is plenty for a trust group; horizontal scaling is a no-op if ever needed because all state is in Postgres or MinIO.

### Worker container

A single Python process running two concurrent loops:

1. **APScheduler** evaluates each active `digest.cadence` (stored as a cron expression in Postgres) and enqueues a `digest_build` job when one is due.
2. **Job loop** pulls jobs from the `jobs` table (`SELECT ... FOR UPDATE SKIP LOCKED`) and executes them: `rss_poll`, `ingest_email_mime`, `summarize_item`, `embed_item`, `digest_build`, `digest_notify`.

Single worker is fine at the Phase 1 scale. Moving to Redis + Arq for multiple workers is a future change that does not require pipeline rewrites.

### Postgres with pgvector

One database. Holds relational data, vector embeddings (1536 or 3072 dims, provider-dependent), and the `jobs` queue. HNSW indexes on the embedding column for k-NN queries. Backups are a single `pg_dump` per schedule.

### MinIO

Self-hosted S3-compatible object store running on the same VPS in Phase 1. Holds:

- Full HTML of RSS items (post-extraction).
- Raw MIME of inbound emails.
- (Later) audio and video blobs.

Migration paths for scale are well-understood (Hetzner Storage Box, Cloudflare R2); no code change beyond a config swap.

### Cloudflare Email Worker

Tiny JavaScript function. Receives inbound mail on the domain's MX, validates DKIM and spam flags, and forwards raw RFC822 as an HTTPS POST to the web container's webhook. Authenticates with `Authorization: Bearer <secret>`. ~30 lines of code.

### Caddy

Existing reverse proxy on the VPS, terminating TLS for all subdomains. Routes `digest.pascalkraus.com` (placeholder) to the web container.

## Data flows at a glance

| Flow                  | Entry                        | Exit                                       |
|-----------------------|------------------------------|--------------------------------------------|
| RSS ingestion         | Scheduled RSS poll job       | New `items` rows, MinIO blobs              |
| Email ingestion       | CF Email Worker → webhook    | New `items` or `pending_inbound` rows      |
| Summarization         | `summarize_item` job         | `items.summary_text` populated             |
| Embedding             | `embed_item` job             | `items.embedding` populated                |
| Digest build          | Scheduled `digest_build` job | New `digest_runs` row with `rendered_json` |
| Digest notification   | `digest_notify` job          | Transactional email sent                   |
| Digest read (web)     | Browser → FastAPI → DB       | Rendered digest HTML                       |
| Digest read (API)     | Agent → FastAPI → DB         | Digest JSON                                |

## What is deliberately not in the architecture

- Redis, Kafka, RabbitMQ, or any dedicated message broker.
- A separate vector database (Qdrant, Weaviate, Pinecone).
- A separate search index (Elasticsearch, Typesense).
- Microservices beyond the web/worker split.
- A self-operated SMTP/MTA for inbound mail.
- Any third-party "auth as a service" provider.

These are all tools that will show up in Phase 4+ if the product ever needs them. Phase 1 does not.
