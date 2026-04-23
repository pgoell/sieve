# Architectural Decision Records

This directory captures the meaningful architectural decisions made for Sieve. Each record is immutable once accepted; changes come via new ADRs that supersede old ones.

## Format

Each ADR follows the Michael Nygard template:

- **Title** (file name `NNNN-kebab-case-title.md`)
- **Status:** Proposed, Accepted, Deprecated, Superseded (with pointer)
- **Context:** what forces are at play
- **Decision:** what we are doing
- **Consequences:** what becomes easier or harder

## Governance

- ADRs are numbered sequentially. Never renumber.
- "Proposed" ADRs are draft; "Accepted" ones are load-bearing.
- If a decision changes, the old ADR gets `Status: Superseded by NNNN` and a link forward. The old ADR is preserved as written, for history.
- The design spec (`docs/superpowers/specs/YYYY-MM-DD-sieve-platform-design.md`) references ADRs; ADRs can reference the spec.
- Before implementing a feature that touches architecture, check ADRs first. If no ADR covers the decision, write one as "Proposed" before coding.

## Index

| # | Title | Status |
|---|-------|--------|
| 0001 | [Architecture shape: API + worker on shared Postgres](0001-architecture-shape.md) | Proposed |
| 0002 | [Shared source catalog with subscription-gated access](0002-shared-catalog-and-subscription-gating.md) | Proposed |
| 0003 | [Inbound email via Cloudflare Email Worker](0003-inbound-email-via-cloudflare-worker.md) | Proposed |
| 0004 | [BYOK LLM keys with optional platform key](0004-byok-with-platform-key-option.md) | Proposed |
| 0005 | [Storage split: Postgres + pgvector for data and vectors, Hetzner Object Storage for blobs](0005-storage-split-postgres-minio.md) | Proposed |

All current ADRs are Proposed. They will be marked Accepted at the start of Phase 0 implementation.
