# ADR 0001: Architecture shape: API + worker on shared Postgres

**Status:** Proposed
**Date:** 2026-04-23

## Context

Sieve has two distinct workloads:

1. **Interactive request/response.** Users browse the web app, fetch digests, read items, manage sources.
2. **Burst-y asynchronous work.** Feed polling, email ingestion, LLM summarization, embedding generation, scheduled digest building.

These have very different latency profiles. LLM summarization can take several seconds; an interactive page load must not. They also have different scaling characteristics: async work benefits from parallelism, the API does not (not at Phase 1 scale).

Three realistic shapes were considered:

1. **Monolith:** one container runs API and async work in the same process, using FastAPI `lifespan` and APScheduler in-process.
2. **API + worker split:** two containers sharing Postgres; the worker consumes a DB-backed job queue.
3. **Full message bus:** API + multiple workers + Redis/RabbitMQ/Streams + separate scheduler.

## Decision

Use shape 2: API container + worker container, shared Postgres.

- The `jobs` table and `SELECT ... FOR UPDATE SKIP LOCKED` provide the job queue.
- APScheduler inside the worker handles cron-like `digests.cadence` evaluation.
- Postgres is the single stateful service for relational data, vectors (via pgvector), and jobs.
- MinIO is the single blob store.

## Consequences

### Easier

- Interactive API latency is never blocked by LLM calls or feed fetches.
- Independent deploy/restart cadence for worker without API downtime.
- One database to back up, snapshot, and restore.
- Clean separation of modules (`lib/sources`, `lib/ingestion`, `lib/digests`, etc.), independently testable.
- Matches the Klassenzeit deployment pattern (GHCR images, Docker Compose, Caddy).

### Harder

- Two containers to operate instead of one. Mitigated by Docker Compose and minimal per-container configuration.
- DB-as-queue has limits around 10s of thousands of jobs per second. Well above Phase 1 scale (hundreds to low thousands per day). If ever needed, move to Redis + Arq without touching the pipeline modules; only the job submit/claim functions change.
- The worker is a single replica in Phase 1. Vertically scalable; horizontal scaling is future work.

### Rejected alternatives

- **Monolith:** one stuck LLM call would slow the API. Worker and API cannot be independently restarted. Acceptable for a weekend prototype, not for a service that Pascal actually uses.
- **Full message bus:** Redis is another stateful service to operate, back up, and restore. Premature for a trust group. Can be introduced later with minimal pipeline changes.
