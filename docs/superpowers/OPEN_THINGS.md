# Open Things

Running log of work items, tech debt, and follow-ups for sieve. Mirrors the shape of the Klassenzeit `OPEN_THINGS.md`: sprint-in-progress sections at the top, then a backlog grouped by concern and ordered by importance within each group. Features, tech debt, and refactorings live together and are ordered the same way; the "quality first, tidy first" rule decides what to pick up next.

Items trace back to the [platform design spec](specs/2026-04-23-sieve-platform-design.md) and the ADRs unless stated otherwise.

## Current sprint

Nothing in progress yet. The first sprint will be Phase 0 (Foundation), at which point items from the backlog's "Phase 0" section move up here as numbered ordered steps.

## Pay down alongside the sprint

Empty until the first sprint begins. Use this section for debt the active sprint will touch, which is cheaper to pay upfront than to retrofit.

## Acknowledged, not in scope this sprint

Empty until a sprint exists. Use this section for items the active sprint brushes against but deliberately leaves alone, to keep scope honest.

## Backlog

Everything below is queued. Ordered roughly by importance within each group. When multiple sections are equally important, prefer the group that unblocks the most downstream work.

### Phase 0: foundation

The invisible prerequisite to Phase 1 shipping any features.

- **Repo scaffold.** Two containers (web, worker), Postgres with pgvector, MinIO, docker-compose for local dev, Dockerfile per service, Caddy subdomain config for staging and production. Match Klassenzeit's layout (`backend/`, `frontend/`, `workers/`, `deploy/`) so the deploy pattern carries over.
- **Toolchain with mise.** `mise.toml` pinning Python, Node, uv, pnpm, cocogitto, lefthook. `mise run dev`, `mise run test`, `mise run lint`, `mise run fmt`, `mise run db:up`, `mise run db:migrate` tasks wired to match Klassenzeit muscle memory.
- **CI in GitHub Actions.** Lint + test + build on every PR; self-hosted runner deploy workflow for staging on master. Mirror `deploy-images.yml` from Klassenzeit so GHCR image publication is identical.
- **Backend package skeleton.** FastAPI app with `lifespan`-managed state (engine, session factory, settings, scheduler handle). SQLAlchemy async, Alembic configured. Pydantic Settings reading from env. Structured JSON logging from day one.
- **Worker container skeleton.** Separate Python entry point, same `src/sieve/` package. APScheduler wired with the Postgres SQLAlchemy job store. Job loop stub with `SELECT ... FOR UPDATE SKIP LOCKED` and a `no_op` job type.
- **Frontend skeleton.** Vite + React + TypeScript + Biome. Empty auth-gated shell. OpenAPI codegen task (`mise run fe:types`) pulling from the backend's emitted schema.
- **Conventional Commits + lefthook + cocogitto.** Commit-msg hook rejecting non-conventional messages. Pre-push running `mise run test`.
- **Secrets bootstrap.** `.env.example` listing every required variable from `docs/architecture/security.md`. Documented in `deploy/README.md`.
- **Postgres with pgvector.** Extension enabled in init migration. A smoke test proving `vector(1536)` columns work.
- **MinIO dev setup.** Local bucket `sieve-content` and `sieve-mime` created on first boot. Scoped app credentials documented.

### Product capabilities (Phase 1)

User-facing features that each ship a visible slice of value.

- **Magic-link auth.** Email a signed short-lived token, redeem via GET link, server-side session cookie. No passwords. Tied to ADR 0002's subscription model: a session user must be resolvable on every request.
- **Invite-code signup.** `POST /redeem?code=...` creates a user account tied to an invite. Invite codes have expiries and single-use flags. Admin can issue invites from the admin UI.
- **Self-service RSS source subscription.** User pastes a feed URL; backend canonicalizes `(kind=rss, identity=url)`, creates or looks up the `sources` row, marks the user's `subscriptions` row as active, queues the first poll. UI: "Add RSS source" form with URL input and health feedback.
- **Admin-gated email source requests.** User submits a request ("please add Ben Thompson's newsletter", with optional sender address hint). Admin UI lists requests and lets the operator create the canonical `email_sender` source and activate the requesting user's subscription.
- **User's alias display.** Settings page shows the user's `<handle>.<random>@in.sieve.pascalkraus.com` with a copy button. Explains the forwarding workflow.
- **BYOK key entry.** Settings page accepts Anthropic and Google API keys. Keys encrypted with Fernet + master key on submit. Submission replaces any existing key.
- **Digest definition UI.** Create / edit / delete digests: name, cadence (preset cron options in Phase 1, custom cron in Phase 2), subscribed sources subset, max items, kind = brief. Each user's digests listed on their dashboard.
- **Digest read view.** Route `/d/<digest_run_id>`. Fetches `digest_runs.rendered_json`, renders item cards: title, source badge, publish time, summary text, "Read here" (inline expand) and "Read original" (external link). Mark-as-read on view or per-item toggle.
- **Notification email.** On `digest_runs` creation, worker sends a short transactional email with subject `"Your <digest.name> is ready"` and a deep link. Outbound provider choice (Resend vs Postmark) is the first decision of this feature's implementation plan.
- **Source list UI.** Per-user view of subscribed sources grouped by kind; pause, unpause, unsubscribe actions. Unsubscribing sets `unsubscribed_at = now()`; items delivered while active remain readable.

### Ingestion: RSS

- **RSS polling scheduler.** Per-source poll interval (default 1h). APScheduler enqueues `rss_poll` jobs when due.
- **Feed fetch with HTTP caching.** Respect `ETag` and `Last-Modified` so repeat fetches are cheap.
- **Feed parse and item upsert.** `feedparser`; upsert via `INSERT ... ON CONFLICT (source_id, external_id) DO NOTHING`.
- **Readable-content extraction.** For each new item, follow the link and extract readable HTML with a library like `readability-lxml` or `trafilatura`. Upload to MinIO as `content/<source_id>/<item_id>.html`.
- **Error handling and health.** 5 retry attempts with exponential backoff on fetch failure; source marked unhealthy in the UI after.

### Ingestion: email

- **Cloudflare Email Worker.** JavaScript function under `workers/inbound-email/`. Deployed via Wrangler. Authenticates to the FastAPI webhook with `Authorization: Bearer <SIEVE_INBOUND_SECRET>`. Runbook in `deploy/`.
- **Inbound webhook handler.** `POST /webhooks/inbound-email` with constant-time bearer check. Parses RFC822, resolves user by `inbound_alias_suffix`, resolves or creates pending `sources` row, stores raw MIME in MinIO, upserts `items` via `(source_id, Message-ID)`.
- **Pending inbound queue.** Mail from unknown senders goes into `pending_inbound`; surfaces a card in the admin UI pre-filled with what we know. On admin approval, queued rows drain into `items`.
- **MIME body extraction.** Parse `multipart/*`, prefer `text/html` body for content, fall back to `text/plain`. Sanitize on read, not on write.
- **Forwarded-newsletter detection.** Some users forward newsletters from a primary inbox; the `From:` then points at themselves. Detect forwarded `From:`/`Reply-To:`/`X-Forwarded-*` headers and use the original sender for source canonicalization.

### Digest pipeline

- **APScheduler with Postgres job store.** Scheduler state survives restarts.
- **`digest_build` job.** Gathers items in the digest's subscriptions since the last run, orders by `fetched_at` desc, caps at `max_items`. See [`docs/architecture/digest-pipeline.md`](../architecture/digest-pipeline.md).
- **Summarization job.** Idempotent; writes `items.summary_text` once per item. Asymmetric BYOK attribution (first subscribed user pays for the summary that is shared with later subscribers) is called out in ADR 0004 and must be preserved.
- **Embedding job.** Idempotent; writes `items.embedding`.
- **`digest_notify` job.** Sends the short transactional email; writes `digest_runs.notified_email_at`.
- **LLM key resolver helper.** `resolve_key(user, provider)` with the BYOK-then-platform-then-error order from ADR 0004. Decrypts BYOK at call time; never caches.
- **`llm_usage` accounting.** Row per call with token and cost attribution; monthly per-user platform-key soft cap.

### Admin

- **Admin flag on users.** Single `is_admin` bool; no separate admin subsystem.
- **Admin dashboard route.** Lists pending email source requests, `pending_inbound` items, and recent `llm_usage`. Same SPA under an admin-only route guard.
- **Source CRUD.** Rename, pause, delete a canonical source. Pausing halts ingestion for all subscribers without severing subscriptions.
- **Invite management.** Issue, revoke, list invite codes.
- **User plan toggle.** Flip `users.plan` between `free` and `paid`. No payment integration in Phase 1.
- **Audit log.** `audit_log` table populated by every admin action. Viewable in admin UI.

### Observability

- **`/metrics` Prometheus endpoint.** On both web and worker containers.
- **Domain counters.** `sieve_items_ingested_total`, `sieve_digest_runs_total`, `sieve_summarize_calls_total`, `sieve_summarize_tokens_total`, `sieve_pending_inbound_size`.
- **Structured logs.** JSON to stdout with `trace_id` threaded from HTTP request into async jobs.
- **OpenTelemetry shim.** Optional exporter defaulting to noop; a real exporter wires in when the VPS-wide Grafana stack exists.

### Testing

- **pytest infrastructure.** `pytest-asyncio`, Postgres container in compose used for integration tests (no mocking of the database; matches Klassenzeit preference).
- **Vitest infrastructure.** Unit tests colocated under `frontend/src/**/__tests__/`.
- **Playwright E2E.** One end-to-end happy-path flow (login, add source, generate digest on demand via a test-only endpoint, read the digest). Extend per feature as they land.
- **Test-only endpoint.** Gated by `SIEVE_ENV=test`, for triggering digest builds on demand and for seeding test data.
- **Inbound webhook test harness.** A fixture that POSTs canned RFC822 messages to `/webhooks/inbound-email` so email ingestion is testable without Cloudflare in the loop.

### Open decisions (surfaced from the spec)

Phase 1 decisions that are intentionally deferred; each unblocks at implementation time.

- **Outbound transactional email provider.** Resend vs Postmark vs SES. Decide when wiring `digest_notify`.
- **Platform-key monthly soft cap amount.** A sensible default (e.g., €5 per paid user per month) plus a per-user override column. Decide before the first paid user.
- **Retention policy default.** 30 vs 90 days vs indefinite for `items` and blobs. Phase 1 can ship with indefinite and revisit in Phase 2.
- **Inbound alias domain.** `in.sieve.pascalkraus.com` vs `in.digest.pascalkraus.com` vs something else. Decide before Cloudflare DNS.
- **Web subdomain.** `sieve.pascalkraus.com` vs another. Decide before Caddy config.
- **Invite code format.** Human-friendly vs opaque random.
- **Frontend component library.** shadcn-style vs raw Tailwind primitives. Decide when the frontend skeleton goes beyond the login form.
- **Worker repo placement.** Keep `workers/inbound-email/` in this repo (preferred) vs a sibling repo.

### Deferred to later phases

Explicitly out of scope for Phase 1; listed so they are not forgotten.

#### Phase 2: archive and semantic search

- Full `/library` UI: browse by source, filter by date, star / archive, keyword search.
- Semantic search: `embed(query) → pgvector k-NN`, optional LLM re-rank, subscription-gated.
- User-configurable retention per source.

#### Phase 3: agent API

- Per-user API tokens.
- Documented REST endpoints: list digests, fetch latest digest JSON, semantic search.
- OpenAPI contract formalized for external consumers.
- Optional MCP wrapper.

#### Phase 4+: the "maybe" pile

- Topic-keyword filters; later natural-language interest profiles.
- Opt-in source suggestions ("you might also like...").
- YouTube and podcast ingestion via Whisper locally or Gemini multimodal.
- Social layer: sharing, comments, recommendations, shared reading lists.
- Additional digest formats (Triage list, Deep Dive).
- Payment flow (Stripe or manual invoice); until then the operator flips `users.plan` by hand.
- Multi-tenant / team accounts.
- Mobile apps.
- Self-hosted local LLM adapter.

### Project metadata

- **License.** Deferred; no `LICENSE` file yet. Revisit when the distribution model (private / open source / SaaS) is clearer.
- **`.claude/CLAUDE.md`.** Port the Klassenzeit workflow rules (quality first, tidy first; TDD; skill invocation rules) once real code starts and the file has something concrete to govern. Keep the bar: nothing gets added to CLAUDE.md that review discipline alone is expected to enforce without a lint rule or test backing it.
- **ADR 0001-0005 status update.** Flip all five from Proposed to Accepted at the start of Phase 0 implementation.
