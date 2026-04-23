# ADR 0002: Shared source catalog with subscription-gated access

**Status:** Proposed
**Date:** 2026-04-23

## Context

Multiple users in the trust group will often subscribe to the same sources (Stratechery, Hacker News RSS, popular newsletters). Two obvious modeling choices:

1. **Per-user sources and items.** Each user has their own rows for every source and every item they receive. Storage is duplicated; items from the same feed are ingested N times.
2. **Shared canonical sources and items.** One `sources` row per `(kind, identity)`, one `items` row per `(source_id, external_id)`. Users are linked to sources via `subscriptions` and to items indirectly.

Choice 2 has two advantages (storage efficiency, catalog discoverability) and one risk (it starts to look like content redistribution if not carefully gated).

## Decision

Use shared canonical sources and items, with explicit access gating.

The gating rules are elevated to non-negotiable platform principles:

- **Subscription-Gated Ingestion.** No content enters the system without at least one user actively subscribing to its source. No speculative crawling.
- **Subscription-Gated Access.** A user can only see items from sources they have an active subscription to, and only items ingested during their active subscription window (`items.fetched_at BETWEEN subscriptions.subscribed_at AND COALESCE(unsubscribed_at, NOW())`).

This pattern is implemented as a single helper that all item-returning queries MUST use. Direct item-by-id lookups also enforce the check.

For email sources specifically, source creation is admin-gated (see ADR 0003), adding another layer of control over what the system will ingest.

## Consequences

### Easier

- Storage scales with unique content, not with users × content. Ten users on Stratechery store one copy, not ten.
- A catalog UI becomes trivially possible: show existing canonical sources users can subscribe to.
- Dedup of identical emails delivered to multiple users' aliases is a primary-key constraint (`UNIQUE (source_id, external_id)`).
- Summaries and embeddings are computed once per item, shared across all subscribed users.

### Harder

- Every query handler must remember to filter through `subscriptions`. Mitigated by a single reusable helper function and a code-review rule.
- Time-bounded access (subscription windows) adds a join condition to item queries. Negligible performance cost, noticeable in query complexity.
- Users who unsubscribe retain access to items delivered during their subscription window. This is intentional (they received those items legitimately) but requires the `unsubscribed_at` column and window logic.

### Legal posture

Shared storage is defensible only because the gating rules above make the system behave, from each user's perspective, exactly like a personal reader: they receive only what they have personally opted into, nothing more. Violating either gating rule transforms the system into something other than a personal reader and would invalidate this posture. The rules are therefore non-negotiable.
