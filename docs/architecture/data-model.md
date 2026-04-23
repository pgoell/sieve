# Data Model

This is the Phase 1 entity reference. Column types are Postgres-style and indicative, not final.

## Entities

### `users`

| Column                 | Type                 | Notes                                                |
|------------------------|----------------------|------------------------------------------------------|
| id                     | uuid PK              |                                                      |
| email                  | citext UNIQUE        | Login identity for magic-link auth                   |
| handle                 | citext UNIQUE        | Short public identifier                              |
| inbound_alias_suffix   | text UNIQUE          | Random component of `<handle>.<random>@in.digest...` |
| plan                   | enum(`free`,`paid`)  | Manual flag for now                                  |
| invited_by             | uuid FK users(id)    | Provenance                                           |
| created_at             | timestamptz          |                                                      |

### `invites`

| Column     | Type                      | Notes                        |
|------------|---------------------------|------------------------------|
| code       | text PK                   | Short, shareable             |
| created_by | uuid FK users(id)         | Issuer                       |
| used_by    | uuid FK users(id) NULL    | Redeemer                     |
| used_at    | timestamptz NULL          |                              |
| expires_at | timestamptz NULL          |                              |

### `user_credentials`

Optional; only present for BYOK users. One row per `(user, provider)`.

| Column            | Type                              | Notes                                   |
|-------------------|-----------------------------------|-----------------------------------------|
| user_id           | uuid FK users(id)                 |                                         |
| provider          | enum(`anthropic`,`google`)        |                                         |
| encrypted_api_key | bytea                             | Fernet-encrypted                        |
| created_at        | timestamptz                       |                                         |
| PK                | (user_id, provider)               |                                         |

### `sources`

Shared canonical catalog. One row per `(kind, identity)`, not per user.

| Column        | Type                         | Notes                                   |
|---------------|------------------------------|-----------------------------------------|
| id            | uuid PK                      |                                         |
| kind          | enum(`rss`,`email_sender`)   |                                         |
| identity      | text                         | RSS URL, or sender email address        |
| display_name  | text                         | Human-friendly; editable by admin       |
| first_seen_at | timestamptz                  |                                         |
| created_at    | timestamptz                  |                                         |
| UNIQUE        | (kind, identity)             |                                         |

### `subscriptions`

Links users to sources. Drives all access checks.

| Column           | Type                                             | Notes |
|------------------|--------------------------------------------------|-------|
| user_id          | uuid FK users(id)                                |       |
| source_id        | uuid FK sources(id)                              |       |
| status           | enum(`pending`,`active`,`paused`,`unsubscribed`) |       |
| subscribed_at    | timestamptz                                      | Access window lower bound |
| unsubscribed_at  | timestamptz NULL                                 | Access window upper bound |
| PK               | (user_id, source_id)                             |       |

### `items`

Canonical items. Dedup by `(source_id, external_id)`.

| Column            | Type                     | Notes                                         |
|-------------------|--------------------------|-----------------------------------------------|
| id                | uuid PK                  |                                               |
| source_id         | uuid FK sources(id)      |                                               |
| external_id       | text                     | RSS GUID/URL or email `Message-ID`            |
| title             | text                     |                                               |
| author            | text NULL                |                                               |
| published_at      | timestamptz NULL         | Publisher's timestamp if available            |
| fetched_at        | timestamptz              | Our ingest time; used for subscription window |
| content_blob_key  | text                     | MinIO key for full content                    |
| summary_text      | text NULL                | Populated by LLM summarization job            |
| embedding         | vector(1536)             | pgvector; dims depend on provider             |
| created_at        | timestamptz              |                                               |
| UNIQUE            | (source_id, external_id) |                                               |

### `item_states`

Per-user read and starred state.

| Column    | Type                | Notes |
|-----------|---------------------|-------|
| user_id   | uuid FK users(id)   |       |
| item_id   | uuid FK items(id)   |       |
| read_at   | timestamptz NULL    |       |
| starred   | bool NOT NULL DEFAULT false |       |
| PK        | (user_id, item_id)  |       |

### `digests`

User-defined digest specs.

| Column            | Type                              | Notes                                            |
|-------------------|-----------------------------------|--------------------------------------------------|
| id                | uuid PK                           |                                                  |
| user_id           | uuid FK users(id)                 |                                                  |
| name              | text                              |                                                  |
| cadence           | text                              | Cron expression                                  |
| subscription_ids  | uuid[] (array of subscriptions PKs) | Which of the user's subs contribute          |
| max_items         | int                               | Cap per run                                      |
| kind              | enum(`brief`)                     | Phase 1 is `brief` only                          |
| created_at        | timestamptz                       |                                                  |

### `digest_runs`

Materialized digests. The `rendered_json` is the canonical payload for web and API.

| Column             | Type                     | Notes                                |
|--------------------|--------------------------|--------------------------------------|
| id                 | uuid PK                  |                                      |
| digest_id          | uuid FK digests(id)      |                                      |
| ran_at             | timestamptz              |                                      |
| item_ids           | uuid[]                   | Ordered as they appear in the digest |
| rendered_json      | jsonb                    | Full payload                         |
| notified_email_at  | timestamptz NULL         |                                      |

### `pending_inbound`

Mail received from senders not yet canonicalized as a source. Awaits admin review.

| Column            | Type                       | Notes                                  |
|-------------------|----------------------------|----------------------------------------|
| id                | uuid PK                    |                                        |
| user_id           | uuid FK users(id)          | Recipient                              |
| sender            | text                       | Parsed `From:`                         |
| raw_mime_blob_key | text                       | MinIO key for the original mail        |
| received_at       | timestamptz                |                                        |

### `llm_usage`

Per-user usage attribution. Drives the platform-key soft cap.

| Column       | Type                              | Notes                                   |
|--------------|-----------------------------------|-----------------------------------------|
| user_id      | uuid FK users(id)                 |                                         |
| period       | text                              | e.g. `2026-04` (monthly bucket)         |
| provider     | enum(`anthropic`,`google`)        |                                         |
| tokens_in    | bigint                            |                                         |
| tokens_out   | bigint                            |                                         |
| cost_cents   | bigint                            |                                         |
| PK           | (user_id, period, provider)       |                                         |

### `jobs`

Durable background job queue.

| Column         | Type               | Notes                                           |
|----------------|--------------------|-------------------------------------------------|
| id             | uuid PK            |                                                 |
| kind           | text               | `rss_poll`, `ingest_email_mime`, etc.           |
| payload_json   | jsonb              |                                                 |
| scheduled_for  | timestamptz        | Not earlier than                                |
| started_at     | timestamptz NULL   |                                                 |
| completed_at   | timestamptz NULL   |                                                 |
| error          | text NULL          |                                                 |
| attempts       | int                |                                                 |

## Access query pattern

Every query that returns items to a user MUST filter through `subscriptions`:

```sql
SELECT i.*
FROM items i
JOIN subscriptions s
  ON s.source_id = i.source_id AND s.user_id = :user_id
WHERE s.status = 'active'
  AND i.fetched_at >= s.subscribed_at
  AND (s.unsubscribed_at IS NULL OR i.fetched_at <= s.unsubscribed_at)
ORDER BY i.fetched_at DESC;
```

This pattern is the implementation of the Subscription-Gated Access principle. It should be encapsulated in a single helper so no handler writes item queries without it.

## Dedup invariants

- RSS: `external_id` is the feed item's `<guid>` if present, else the item URL. Feed polling MUST use `INSERT ... ON CONFLICT (source_id, external_id) DO NOTHING`.
- Email: `external_id` is the `Message-ID` header. The same email delivered to multiple users' aliases is ingested once; multiple `subscriptions` give multiple users access to the same `items` row.

## Migrations

Alembic. Per Klassenzeit convention, one migration per schema change, reviewed and squashed only in exceptional cases.
