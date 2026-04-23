# Sieve Platform Design

**Date:** 2026-04-23
**Status:** Proposed
**Audience:** implementers, future Claude sessions, Pascal

---

## 1. Summary

Sieve is a scheduled digest platform: a user subscribes to content sources (RSS feeds and emailed newsletters in Phase 1; YouTube and podcasts later), and the system produces personalized briefs on a schedule the user controls. Each brief pulls items from the user's chosen sources, summarizes them via an LLM, and links to originals. The web app is the primary consumption surface; email delivers a short notification with a deep link; a JSON API makes the same digest consumable by agents.

The product is scoped for a small trusted group, invite-only, with admin curation of email sources (to keep out of paid-content redistribution risk). The architecture is an API + worker split on shared Postgres with pgvector for embeddings and a MinIO object store for content blobs.

---

## 2. Audience and scope

### Who

A small, trusted group of known users. Invite-only signup via admin-issued codes. Not a public product. The trust model assumes users are known to the operator and will not abuse the system.

### Why the scope matters

A public product would force different decisions on auth, abuse handling, content licensing, and billing. Most critically, emailed newsletters and transcribed podcast content are copyrighted material. For a trust group, the platform looks like a fancy shared RSS/email reader; for a public product, it starts to look like a content redistributor. The legal posture drives all the subscription-gating rules in Section 10.

### Out of scope

- Public signup, trials, pricing tiers beyond a manual free/paid flag.
- Federated or decentralized ingestion.
- Rich editorial content produced by the platform itself.
- Moderation or community governance beyond an invite list.

---

## 3. Design principles

Every design decision in this spec is downstream of the following invariants. Violating any of them is a spec change, not an implementation choice.

1. **User Sovereignty.** The user is in control of what ends up in their feed. The system never silently adds sources, auto-includes items in digests, or ranks content by undisclosed algorithms. Suggestions are opt-in surfaces, not pipeline steps.
2. **Subscription-Gated Ingestion.** No content enters the system without at least one user actively subscribing to its source. No speculative crawling.
3. **Subscription-Gated Access.** A user can only see items from sources they have an active subscription to, and only items ingested during their active subscription window.
4. **BYOK as default, platform key as convenience.** LLM cost attribution is always explicit per user. Platform-key usage is capped.
5. **Structured first, rendered second.** A digest is a JSON document; the web app and the agent API read the same payload.
6. **Observability is a property, not a feature.** The service emits Prometheus metrics and structured JSON logs from day one.

---

## 4. Phasing

The vision is intentionally larger than a sensible first release. The repo captures the full vision; the first implementation plan attacks a tight slice.

### Phase 0: Foundation

Repo scaffolded in the Klassenzeit shape. CI, lint, test, mise, Docker, Caddy subdomain, Postgres + pgvector + MinIO wired up, Alembic migrations, magic-link auth with invite codes working, empty React shell behind auth. No user-facing features yet. The point is that Phase 1 ships features, not plumbing.

### Phase 1 (MVP): RSS + email Brief

The narrowest end-to-end slice that delivers value.

- Invited user signs in (magic link)
- Adds their Anthropic key (BYOK), or is on a paid plan with access to the platform key
- RSS: subscribes to sources by URL (self-service; auto-creates the canonical source if new)
- Email: submits a request for a newsletter; admin adds the canonical source; user forwards or signs up with their alias
- Worker polls RSS, receives email via Cloudflare Email Worker webhook, ingests items (dedup via `(source_id, external_id)`), stores full content in MinIO, generates embeddings
- User creates one or more digests (`name`, `cadence`, `source subset`, `max_items`, `kind=brief`)
- On schedule, worker gathers recent items, summarizes via user's key, assembles digest JSON, persists `digest_runs`, sends a short email notification with a deep link
- Web app renders the digest and supports inline expansion of item content; marks items as read

Explicitly **not** in Phase 1: full archive/library UX, semantic search, agent API, topic filters, interest profiles, social, YouTube, podcasts.

### Phase 2: Archive and semantic search

- `/library` UI: browse by source, filter by date/source, mark starred, keyword search
- Semantic search endpoint: question → embedding → pgvector k-NN → optional LLM re-rank
- User-configurable retention policy per source

### Phase 3: Agent API

- Per-user API tokens
- Documented endpoints: list digests, fetch latest digest JSON, semantic search
- OpenAPI contract formalized for external consumers
- Optional: MCP server wrapper around the REST API

### Phase 4+: the "maybe" pile

Not scoped here, listed for completeness:

- Topic-keyword filters; later natural-language interest profiles
- Opt-in source suggestions
- YouTube and podcast ingestion (Whisper locally or Gemini multimodal)
- Social layer: sharing, comments, recommendations between users
- Additional digest formats (Triage, Deep Dive)

---

## 5. Architecture

### Shape

API + worker split, shared Postgres, same-host object store. Four long-running containers.

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

### Components

- **Web container.** FastAPI serving the REST API, the webhook endpoints, and the built React SPA's static assets. Pure synchronous request/response. No long-running work.
- **Worker container.** A single Python process running APScheduler (cron-like per-digest schedules stored in Postgres) plus a job loop that reads from a `jobs` table using `SELECT ... FOR UPDATE SKIP LOCKED`. Handles feed polling, email ingestion follow-through, content normalization, summarization via BYOK or platform LLM key, embedding generation, and digest rendering.
- **Postgres.** Backbone. Stores relational data, pgvector embeddings, and the job queue. One backup, one snapshot, one restore path.
- **MinIO.** S3-compatible object store for heavy blobs: full HTML content, raw MIME, later audio and video. Self-hosted on the VPS in Phase 1; migration path to Hetzner Storage Box or Cloudflare R2 if storage or egress grows.
- **Cloudflare Email Worker.** Tiny JavaScript function on `*@in.digest.pascalkraus.com`. Receives inbound mail, validates, POSTs raw RFC822 to the FastAPI webhook with a shared-secret bearer token. Lives either in a `workers/inbound-email/` subdirectory of this repo or a small sibling repo.
- **Caddy.** Existing reverse proxy, handles TLS, routes `digest.pascalkraus.com` (placeholder subdomain) to the web container.

### Why this shape

- Clean separation between burst-y async work (summarization, ingestion) and user-facing request/response. A stuck LLM call cannot stall the API.
- Four containers is the same order of complexity as Klassenzeit; mental model carries over.
- No additional stateful services beyond Postgres and MinIO. Redis, Kafka, and a dedicated message broker are explicit non-goals for Phase 1.
- DB-as-queue (`SELECT ... FOR UPDATE SKIP LOCKED`) is genuinely fine at this scale. If load grows, the queue can move to Redis + Arq without touching the pipeline modules.

### Stack

| Layer      | Choice                                              | Rationale                                           |
|------------|-----------------------------------------------------|-----------------------------------------------------|
| Backend    | Python + FastAPI + SQLAlchemy + Alembic             | Matches Klassenzeit; mature, typed, fast to build  |
| Frontend   | React + Vite + Biome + TypeScript                   | Matches Klassenzeit; OpenAPI codegen for types      |
| Database   | PostgreSQL 17 with pgvector                         | One system for relational + vector search           |
| Object store | MinIO (later Hetzner Storage Box or R2)           | Self-hosted; S3 API; cheap                          |
| Toolchain  | mise for pinning Rust/Python/Node/uv/pnpm           | Klassenzeit convention                              |
| Deploy     | GHCR images + Docker Compose + Caddy                | Klassenzeit convention                              |
| Inbound email | Cloudflare Email Worker + webhook                | No SMTP/MX ops burden                               |
| Outbound email | Resend or Postmark (TBD in Phase 1)              | Transactional only; small volume                    |
| Scheduling | APScheduler                                         | In-process cron in the worker                       |
| LLM        | Anthropic + Google Gemini via pluggable interface   | BYOK + optional platform key                        |

---

## 6. Data model

The v1 entity set, kept deliberately small. Full details in [`docs/architecture/data-model.md`](../../architecture/data-model.md).

```
users              (id, email, handle, inbound_alias_suffix,
                   plan [free|paid], invited_by, created_at)
invites            (code, created_by, used_by, used_at, expires_at)
user_credentials   (user_id, provider, encrypted_api_key, created_at)
                   -- optional; only present for BYOK users

sources            (id, kind [rss|email_sender],
                   identity [rss url | sender address],
                   display_name, first_seen_at, created_at)
                   UNIQUE (kind, identity)
                   -- canonical, shared across users

subscriptions      (user_id, source_id, status [pending|active|paused|unsubscribed],
                   subscribed_at, unsubscribed_at,
                   PK (user_id, source_id))

items              (id, source_id, external_id, title, author,
                   published_at, fetched_at,
                   content_blob_key, summary_text, embedding,
                   created_at)
                   UNIQUE (source_id, external_id)
                   -- canonical, shared; dedup by source + message/guid

item_states        (user_id, item_id, read_at, starred, PK (user_id, item_id))

digests            (id, user_id, name, cadence,
                   subscription_ids [array], max_items, kind [brief],
                   created_at)

digest_runs        (id, digest_id, ran_at, item_ids [array],
                   rendered_json, notified_email_at)

pending_inbound    (id, user_id, sender, raw_mime_blob_key, received_at)

llm_usage          (user_id, period, provider,
                   tokens_in, tokens_out, cost_cents)

jobs               (id, kind, payload_json, scheduled_for,
                   started_at, completed_at, error, attempts)
```

### Key modeling decisions

- **Sources are a shared, canonical catalog.** One row per `(kind, identity)`. Ten users subscribed to Stratechery means one `sources` row, one `items` row per article, one blob per item in MinIO.
- **Access is gated by subscription, not by data shape.** All queries that return items to a user must join through `subscriptions` and respect `subscribed_at` and `unsubscribed_at`. Un-subscribing does not retroactively revoke access to already-delivered items (those remain readable in the user's past digests and archive); it stops new items from flowing.
- **Items are deduplicated per `(source_id, external_id)`.** For RSS, `external_id` is the feed's GUID or item URL. For email, it is the `Message-ID` header. If the same email arrives at multiple users' aliases, the second delivery is idempotent.
- **Embeddings live next to the items.** pgvector column directly on `items`. No separate vector store.
- **Blobs live in MinIO.** `content_blob_key` is the S3 key for the full content. Postgres never stores large text or binary.
- **Jobs are a table, not Redis.** `SELECT ... FOR UPDATE SKIP LOCKED` gives us a real queue with retries and backoff without another stateful service.

---

## 7. Key workflows

Detailed flows in [`docs/architecture/ingestion.md`](../../architecture/ingestion.md) and [`docs/architecture/digest-pipeline.md`](../../architecture/digest-pipeline.md). Sketches here.

### RSS ingestion

1. Worker ticks each active RSS source at its configured poll interval (default 1h).
2. Fetches the feed, diffs against stored `items.external_id`.
3. For each new item: follows the link, extracts readable content, stores HTML blob in MinIO, creates the `items` row.
4. Queues a job to generate summary and embedding.
5. Downstream summary/embedding job executes using the owning user's resolved LLM key.

### Email ingestion

1. User forwards or subscribes with their alias (`<handle>.<random>@in.digest.pascalkraus.com`).
2. Cloudflare Email Worker receives the mail, validates DKIM/spam flags, and POSTs the raw RFC822 to `POST /webhooks/inbound-email` with a shared-secret bearer token.
3. FastAPI handler parses `To:` to resolve the recipient user by `inbound_alias_suffix`, parses `From:` to resolve or create the canonical `source`.
4. If the sender matches a known `source` and the user has an active subscription: create the item (idempotent via `Message-ID`), store MIME blob, queue summary+embedding job.
5. If the sender is unknown: store the raw MIME in `pending_inbound` and notify the user that a request to add this sender is pending admin review.
6. Admin dashboard lets the operator promote pending senders into canonical sources, at which point queued `pending_inbound` rows become `items`.

### Digest generation

1. APScheduler evaluates each active digest's `cadence` every minute.
2. When a digest is due: worker gathers items belonging to its subscribed sources since the last `digest_run`, orders by recency, caps at `max_items`.
3. For each selected item: if a summary exists, reuse; otherwise call the user's LLM provider via BYOK or platform key, store summary back on the item.
4. Assemble digest JSON (items, summaries, metadata, run timestamp, deep links).
5. Persist `digest_runs.rendered_json`.
6. Send notification email via transactional provider with deep link to `digest.pascalkraus.com/d/<digest_run_id>`.

### User happy path

1. Invited user opens invite link → magic-link login → logged in.
2. Settings: pastes Anthropic key, or is on a paid plan.
3. Sources: subscribes to a known RSS source (or pastes a URL, which auto-creates the canonical source).
4. Sources: submits a request to add a new email newsletter; admin approves in the admin UI.
5. User signs up with the newsletter at `pascal.a7x2k@in.digest.pascalkraus.com`.
6. Creates a Digest: `Morning Brief`, cadence `0 7 * * *`, all sources, `max_items=10`, `kind=brief`.
7. Next morning: receives a short notification email with a deep link.
8. Opens the web app, reads the brief in about five minutes, expands one or two items inline, clicks through on one to the original.

---

## 8. LLM usage and BYOK

### Providers

Anthropic and Google Gemini. Interfaces (`Summarizer`, `Embedder`) are abstract; implementations are swappable. Design never assumes a single provider.

### Key resolution

At each LLM call site:

```
resolve_key(user, provider):
  if user has BYOK for provider:
      return user's BYOK (decrypted)
  elif user.plan == 'paid':
      return platform key (from env)
  else:
      raise NoKeyAvailable
```

Platform keys live in server env, never in `user_credentials`. BYOK is encrypted at rest with Fernet and a server-side master key.

### Budget and usage

Every LLM call writes a row to `llm_usage` with token counts and cost. Platform-key users have a monthly soft cap (specific amount TBD in Phase 1). BYOK users are not capped by the platform; their bill goes to their provider directly.

### Where LLMs are used

Phase 1: summarization only (one summary per item, reused across digests that include the item). Embeddings are generated for every item to enable Phase 2 semantic search, using the ingesting user's key (or platform key fallback for paid users). Phase 2+ introduces re-ranking and possibly tagging.

### Why this design

- Cost is always attributable; a runaway digest cannot blow up an untracked bill.
- Swapping providers is a matter of writing a new adapter, not rewriting the pipeline.
- Local inference (Ollama, llama.cpp) is a future adapter implementation, not an architectural change.

---

## 9. Email ingestion specifics

### Inbound path

`Cloudflare Email Worker` on `*@in.digest.pascalkraus.com`:

1. Receives the inbound message.
2. Rejects messages failing DKIM if configured strictly (configurable).
3. Sends raw RFC822 as the body of an HTTPS POST to `https://digest.pascalkraus.com/webhooks/inbound-email` with `Authorization: Bearer <SIEVE_INBOUND_SECRET>`.
4. Returns success to the MTA; no mailbox to poll, no credentials to rotate.

### Alias scheme

Each user gets `<handle>.<random>@in.digest.pascalkraus.com`. The `<random>` is a short high-entropy token; the full alias is the user's to hand out. The `<handle>` exists for human readability in `To:` inspection; the `<random>` is what we actually index against.

### Admin-gated source creation for email

Email sources are only created via an admin approval step. Users submit a "please add sender X" request in the UI. The admin reviews and creates the canonical source. This is the gate we use to keep paid-content redistribution off the platform: if a user wants a paid newsletter, they are expected to subscribe to it themselves and forward to their alias; the platform only ingests what lands legitimately in their alias.

If mail arrives from a sender that is not yet a canonical source: we park the raw MIME in `pending_inbound` and surface a request card in the admin UI, pre-filled with what we know (sender, sample subject, sample content snippet). The admin promotes or rejects.

### Outbound (notifications)

Simple transactional email only: "Your Morning Brief is ready: <deep link>". No digest content in email. Vendor: Resend or Postmark, decided in Phase 1. Outbound reliability and deliverability are not critical for a short link; we can accept the occasional missed notification without losing data, because the digest is always accessible on the web.

---

## 10. Security

Full threat model and mitigation list in [`docs/architecture/security.md`](../../architecture/security.md). Highlights:

- **Auth:** magic-link only. No passwords. Invite-only signup via admin-issued codes. Short-lived signed login tokens.
- **Secrets:** BYOK API keys encrypted at rest with Fernet + server-side master key loaded from env. Master key in env only, never in code or logs.
- **Inbound webhook:** authenticated by a shared secret matched by constant-time comparison. Rejects without.
- **Rate limiting:** on auth endpoints and the inbound webhook. Per-IP and per-account.
- **Transport:** HTTPS only. HSTS via Caddy. No HTTP fallback.
- **SQL:** parameterized queries everywhere; SQLAlchemy ORM enforces this.
- **CSRF:** same-origin plus short-lived session cookies on the SPA.
- **Legal:** the subscription-gating rules in Section 3 are the primary copyright-risk mitigation.

---

## 11. Observability

Sieve emits the following from day one and does not assume a specific monitoring stack:

- **Metrics:** Prometheus on `/metrics` for the web and worker containers. Standard process metrics plus domain counters: items ingested per source, digest runs, LLM calls with token counts, job queue depth.
- **Logs:** structured JSON to stdout. Trace IDs propagated through web requests and into worker jobs.
- **Tracing:** optional OpenTelemetry export, defaulting to noop.

The VPS-level stack (Prometheus + Grafana + Loki or similar) is managed outside this repo; Sieve plugs into whatever exists. Setting up that stack is its own project and is not blocking for Phase 1.

---

## 12. Repo layout (future implementation)

This repo is currently docs-only. When implementation begins, the expected shape (following Klassenzeit conventions) is:

```
sieve/
├── README.md
├── docs/                          # this directory; stays as is
├── backend/                       # FastAPI app
│   ├── src/sieve/
│   │   ├── api/                   # HTTP handlers, deps
│   │   ├── sources/               # feed adapters: rss, email_imap (unused), cloudflare_webhook
│   │   ├── ingestion/             # normalize, dedupe, persist
│   │   ├── content/               # extraction, blob storage
│   │   ├── llm/                   # Summarizer, Embedder interfaces + adapters
│   │   ├── digests/               # scheduler, builder
│   │   ├── search/                # embedding queries, re-rank (Phase 2)
│   │   ├── delivery/              # email notifier, API serializers
│   │   ├── auth/                  # magic-link, invites
│   │   └── admin/                 # source approval UI endpoints
│   ├── alembic/
│   ├── tests/
│   └── pyproject.toml
├── frontend/                      # React + Vite SPA
│   ├── src/
│   ├── tests/
│   └── package.json
├── workers/
│   └── inbound-email/             # Cloudflare Email Worker
├── deploy/
│   ├── compose.production.yaml
│   └── README.md                  # deployment runbook
├── mise.toml
└── CONTRIBUTING.md
```

Each module in `backend/src/sieve/` sits behind a small interface. Tests are colocated under `backend/tests/` mirroring the module tree. The frontend uses OpenAPI codegen from the backend's emitted schema, same as Klassenzeit.

---

## 13. Open questions

The following are intentionally unresolved at spec time; they will be settled in the Phase 1 implementation plan or later.

- **Outbound email provider.** Resend vs Postmark vs SES. Not blocking; all three work.
- **Specific retention defaults.** 30 days, 90 days, indefinite. User-configurable in Phase 2; needs a Phase 1 default.
- **Platform-key monthly cap amount.** Per paid user. Start conservative.
- **Frontend component library.** shadcn-style vs raw Tailwind components. Irrelevant at spec; decide during Phase 0 scaffold.
- **Invite code format.** Human-friendly vs opaque. Minor.
- **Alias subdomain vs prefix.** `*@in.digest.pascalkraus.com` vs `*+sieve@...` style. Leaning subdomain.
- **Worker repo placement.** Inside `workers/` in this repo vs a sibling `sieve-inbound-worker` repo. Leaning same repo.
- **Payment flow (when it arrives).** Stripe vs manual invoice. Out of scope until a user actually pays.

---

## 14. Explicitly deferred

Listed so future readers know these were considered and pushed out on purpose:

- Natural-language interest profiles for filtering inside a digest.
- Topic-keyword classification and filtering.
- YouTube and podcast ingestion (transcription pipeline).
- Social layer (sharing, comments, recommendations).
- Semantic search UI (design has the columns in place; UI is Phase 2).
- Agent API (data contract is implicit from Phase 1; explicit in Phase 3).
- Multiple digest formats (Triage list, Deep Dive).
- Multi-tenant / team accounts.
- Mobile apps.
- Self-hosted local LLM adapter (the interface accommodates it; adapter is future work).
