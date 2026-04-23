# Digest Pipeline

A digest is a user-owned spec that produces a `digest_run` on a schedule. A `digest_run` is a frozen, structured JSON document that the web app renders and the agent API serves. There is no mutation of a past digest; corrections happen in the next run.

## Digest specs

Users create digests with:

- `name` (e.g. "Morning Brief")
- `cadence` (cron expression)
- `subscription_ids` (a subset of the user's active subscriptions that contribute to this digest)
- `max_items` (cap per run)
- `kind` (Phase 1: `brief` only)

A user can have many digests. One user could have a daily brief of AI sources and a weekly digest of design writing; both draw from the same underlying items.

## Scheduling

The worker runs APScheduler. A scheduler tick (every minute) evaluates each active digest's `cadence` against the current time and the last `digest_runs.ran_at`. When a digest is due, the scheduler enqueues a `digest_build` job.

APScheduler state lives in Postgres (via its SQLAlchemy job store) so scheduler state survives restarts.

## Build

```
┌──────────────────────┐
│  digest_build job    │
└──────────┬───────────┘
           │
           ▼
  resolve digest spec + owning user
           │
           ▼
  compute window: since last digest_runs.ran_at (or created_at if first run)
           │
           ▼
  gather candidate items:
    SELECT items through subscriptions where
      subscription_id IN digest.subscription_ids AND
      item.fetched_at > window_start AND
      item.fetched_at <= now()
           │
           ▼
  order by item.fetched_at DESC, cap at max_items
           │
           ▼
  for each selected item:
    if items.summary_text IS NULL:
      call resolve_key(user, provider) and summarize
      persist items.summary_text
      update llm_usage
           │
           ▼
  assemble rendered_json:
    {
      "digest_id": ...,
      "run_id": ...,
      "generated_at": ...,
      "items": [
        {
          "id": ..., "source": {...}, "title": ..., "url": ...,
          "summary": ..., "published_at": ..., "read_url": ...
        }, ...
      ],
      "metadata": { "window_start": ..., "window_end": ..., "item_count": ... }
    }
           │
           ▼
  INSERT digest_runs
           │
           ▼
  enqueue digest_notify job
```

## Deliver

```
┌────────────────────────┐
│  digest_notify job     │
└──────────┬─────────────┘
           │
           ▼
  resolve user + transactional email provider
           │
           ▼
  send short email:
    Subject: "Your <digest.name> is ready"
    Body:    "<digest.item_count> items ready. Read: <deep link>."
           │
           ▼
  write digest_runs.notified_email_at
```

The email never contains digest content. This is a deliberate simplification:

- Email rendering is hard (Outlook, Gmail clipping, dark mode, image blocking).
- The canonical surface is the web app.
- A missed email is a missed notification, not missed content. The user can always visit the app.

## Read

### Web

Browser requests `/d/<digest_run_id>`. FastAPI loads the `digest_runs.rendered_json` and passes it to the SPA. The SPA renders from the same JSON shape the API will serve in Phase 3.

### API (Phase 3)

`GET /api/v1/digests/<digest_id>/runs/latest` → `digest_runs.rendered_json`, authenticated with a per-user API token.

The web render and the API response are the same JSON. No divergent serializers.

## Inline content expansion

When a user expands an item in the digest view, the SPA fetches `/api/items/<item_id>/content`. The handler:

1. Verifies the user has an active subscription to the item's source (and the item was fetched within the subscription window).
2. Loads the content blob from MinIO using `items.content_blob_key`.
3. Returns sanitized HTML.

HTML sanitization is a required security step; see [security.md](security.md).

## Re-runs and edits

Digests are immutable once materialized. To change a digest:

- Editing the spec (e.g. `cadence`, `max_items`) affects future runs only.
- Deleting a digest deletes the spec and hides its past `digest_runs` from the UI; the rows are retained for audit and can be hard-deleted on request.
- There is no "rebuild past digests" operation.

## Observability points

- `sieve_digest_runs_total{kind, result}` counter
- `sieve_digest_build_duration_seconds{kind}` histogram
- `sieve_digest_items_per_run` histogram (how many items landed in each run)
- `sieve_summarize_calls_total{provider, result}` counter
- `sieve_summarize_tokens_total{provider, direction}` counter (in/out)
- Structured log per digest run with `trace_id`, `digest_id`, `user_id`, item count, token counts
