# Vision

## What Sieve is

A platform where a user logs in to curate their own personalized newsletter digest. Sieve ingests newsletters, blogs, RSS feeds, and eventually YouTube and podcasts. The user picks what they want in each digest, at what depth, at what cadence. Each digest is a short brief that filters the relevant stuff, highlights the insights, and points the user at the original items they may want to read in full.

The website also acts as a reader and an archive for ingested content. The same content is exposed as a structured JSON API so agents can consume digests and search the archive.

## Who it is for

A small, trusted group of known users. Invite-only. No public signup. The legal and operational posture is that of a personal tool shared with friends, not a public content platform.

## Why it exists

Reading by subscription has gotten good (there are hundreds of great newsletters, blogs, and podcasts), but consumption has not. Inboxes fill up faster than they can be read. Context is scattered across tools. The user ends up either ignoring most of what they subscribed to, or spending more time triaging than reading.

Sieve is the triage and reading layer. It fetches everything you subscribed to, keeps it, summarizes it, delivers the summaries on a schedule you control, and gets out of your way. You keep reading originals when they are worth reading.

## Guiding principles

Sieve is opinionated about what it will and will not do. The following invariants are non-negotiable and shape every design decision.

1. **User Sovereignty.** The user is in control of what ends up in their feed. The system can suggest, but never silently adds sources, auto-includes items in digests, or ranks content by undisclosed algorithms. Recommendations are opt-in surfaces, never pipeline steps.
2. **Subscription-Gated Ingestion.** No content enters the system without at least one user having actively subscribed to its source. There is no speculative crawling.
3. **Subscription-Gated Access.** A user can only see items from sources they have subscribed to, and only items ingested during their active subscription window.
4. **BYOK as default, platform key as convenience.** Every LLM call is attributable to a user. The platform key is a paid convenience, not a shared resource.
5. **Structured first, rendered second.** A digest is a JSON document before it is HTML or an email. The web app and the agent API read the same payload.

## Success criteria

Sieve is succeeding when:

- The user opens their daily Brief, reads it in under five minutes, and closes it feeling caught up.
- The user's email inbox no longer contains newsletter clutter. Newsletters go to their alias; briefs are the only relevant mail they receive from the platform.
- An agent (e.g. Claude Code) can authenticate and fetch the latest digest as JSON in one request.
- The user trusts that nothing is in their digest they did not explicitly sign up for.

## Long arc (not MVP)

Over time, Sieve will grow to support:

- Topic keyword filters and natural-language interest profiles, both as optional refinements over the base "everything from these sources" selection.
- YouTube and podcast ingestion with transcription (via Whisper locally for the self-hosted path, or Gemini multimodal as an alternative).
- A social layer for the trust group: sharing items, recommending sources to other users, discussion threads on items.
- Full-text and semantic search over the archive.
- Configurable digest formats beyond Brief (a lightweight Triage list; a weekly Deep Dive with cross-source synthesis).

These are captured in the design spec as later phases, not Phase 1.
