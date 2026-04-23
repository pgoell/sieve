# Ingestion

Two source kinds in Phase 1: RSS and email. Both flow through the same post-ingest pipeline (normalize, store blob, create `items` row, queue summary and embedding jobs). Only the front ends differ.

## RSS ingestion

### Self-service source creation

Users add RSS sources directly in the UI by pasting a feed URL. If a canonical `sources` row already exists for that URL, the user is subscribed to the existing source. If not, a new canonical source is created and the user is subscribed.

Admin can later rename, pause, or delete a canonical RSS source. User subscriptions survive renames.

### Polling

Each active RSS source has a poll interval (default 1h, configurable per source). A scheduler loop on the worker enqueues `rss_poll` jobs when polls are due.

### Flow

```
┌──────────────┐
│ rss_poll job │
└──────┬───────┘
       │
       ▼
  fetch feed (GET; HTTP caching headers)
       │
       ▼
  parse (feedparser)
       │
       ▼
  for each entry:
    ├── compute external_id = guid or link
    ├── INSERT items ... ON CONFLICT (source_id, external_id) DO NOTHING
    ├── if inserted:
    │     ├── fetch link → extract readable content
    │     ├── upload HTML to object store → set content_blob_key
    │     ├── enqueue summarize_item job
    │     └── enqueue embed_item job
    └── else:
          skip (already have it)
```

### Error handling

- Feed fetch failures: logged, retried with exponential backoff up to 5 attempts, then alert.
- Parse failures: logged with the offending payload in the object store for debugging, source marked unhealthy in UI.
- Content extraction failure: item still created with `content_blob_key = NULL`; summarization can still use the feed's `<description>`.

## Email ingestion

### Admin-gated source creation

Users submit requests in the UI ("please add Ben Thompson's newsletter"). Requests include the sender address or newsletter name. Admin reviews and creates the canonical `email_sender` source, at which point the user's subscription transitions from `pending` to `active`.

The gate exists to keep paid-content redistribution off the platform. For public free newsletters, approval is a formality; for anything paid, the admin can decline or attach conditions.

### Cloudflare Email Worker

On the domain's MX, `*@in.digest.pascalkraus.com` routes to a Cloudflare Email Worker. The Worker:

1. Receives the inbound email.
2. Optionally checks DKIM/SPF/spam flags.
3. POSTs the raw RFC822 to `https://digest.pascalkraus.com/webhooks/inbound-email` with `Authorization: Bearer <SIEVE_INBOUND_SECRET>`.
4. Returns `2xx` to the sending MTA on Worker success.

Worker code lives in `workers/inbound-email/` in the sieve repo. About 30 lines. Deployed via Wrangler; deployment runbook is part of the deploy docs.

### Flow (webhook handler side)

```
POST /webhooks/inbound-email
       │
       ▼
  verify Authorization bearer token (constant-time compare)
       │
       ▼
  parse To: header → resolve user by inbound_alias_suffix
       │
       ▼
  parse From: header → resolve sender address
       │
       ▼
  lookup sources where (kind='email_sender', identity=sender)
       │
       ├── found AND user has active subscription:
       │        │
       │        ▼
       │   INSERT items (source_id, external_id=Message-ID, ...)
       │     ON CONFLICT DO NOTHING
       │        │
       │        ├── upload raw MIME to object store → content_blob_key
       │        ├── extract HTML body → upload to object store (optional secondary key)
       │        ├── enqueue summarize_item
       │        └── enqueue embed_item
       │
       ├── found AND user's subscription is 'pending':
       │        │
       │        ▼
       │   pending_inbound row (hold until sub becomes active)
       │
       └── not found (sender unknown):
                │
                ▼
           pending_inbound row
                │
                ▼
           notify admin: "new sender to <user>, review?"
```

### Idempotency

Inbound mail MUST be idempotent. If Cloudflare retries, or the same email is delivered to multiple users' aliases, the `(source_id, external_id=Message-ID)` unique constraint ensures one `items` row per message. Access to that item is per-user via `subscriptions` and `item_states`.

### Handling pending admin review

When a sender is unknown, we park the raw MIME in `pending_inbound` and surface it in the admin UI. On admin approval:

1. Canonical `sources` row is created.
2. The requesting user's subscription is flipped from `pending` to `active`, with `subscribed_at = now()`.
3. Queued `pending_inbound` rows are drained into `items` and their downstream jobs.

On admin rejection:

1. `pending_inbound` rows are deleted (or soft-deleted for audit).
2. The user's subscription is marked `unsubscribed` with a reason.
3. The user is notified.

## Shared post-ingest pipeline

Regardless of how an item arrived, after the `items` row exists:

```
enqueue summarize_item(item_id):
  - look up item and owning user(s) via subscriptions
  - resolve LLM key (BYOK → platform key → error)
  - call provider (Anthropic or Google) with the item content
  - write items.summary_text
  - update llm_usage

enqueue embed_item(item_id):
  - resolve LLM key
  - call provider embeddings endpoint
  - write items.embedding
  - update llm_usage
```

Summaries and embeddings are generated once per item and shared across all subscribed users, even if those users used different keys. For Phase 1, the key used is the first subscribing user's key (or platform key, if that user is paid). This is a pragmatic cost-attribution choice; it is explicitly called out in the design because it is slightly asymmetric.

## Observability points

Every ingestion path emits:

- `sieve_items_ingested_total{source_kind, result}` counter
- `sieve_ingest_duration_seconds{source_kind}` histogram
- `sieve_pending_inbound_size` gauge
- Structured log line per item with `trace_id`, `source_id`, `external_id`, and outcome
