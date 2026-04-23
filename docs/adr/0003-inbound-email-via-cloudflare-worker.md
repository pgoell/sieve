# ADR 0003: Inbound email via Cloudflare Email Worker

**Status:** Proposed
**Date:** 2026-04-23

## Context

Email newsletter ingestion is in Phase 1 scope. Each user gets a unique inbound alias (`<handle>.<random>@in.digest.pascalkraus.com`); inbound mail must reach the FastAPI worker for parsing and storage.

Three practical mechanisms exist:

1. **IMAP polling of a third-party inbox.** All mail is forwarded to a single mailbox (Fastmail, mailbox.org); the worker polls IMAP. Requires credentials, has polling latency, pins ingestion to one upstream inbox.
2. **Self-hosted Postfix on the VPS.** Max control; significant ops burden (DKIM, SPF, DMARC, PTR records, spam handling, bounces, reputation).
3. **Inbound-email-as-a-service via Cloudflare Email Workers.** A small JavaScript function on the Cloudflare edge receives mail for the domain and POSTs raw RFC822 to an authenticated webhook. Free at this volume, no SMTP operational burden.

The domain is already behind Cloudflare, making option 3 a natural extension of existing infrastructure rather than a new vendor.

## Decision

Use Cloudflare Email Workers for inbound email:

1. `*@in.digest.pascalkraus.com` has an MX record pointing to Cloudflare.
2. An Email Worker receives each message, optionally inspects DKIM/SPF/spam flags, and POSTs the raw RFC822 to `https://digest.pascalkraus.com/webhooks/inbound-email` with `Authorization: Bearer <SIEVE_INBOUND_SECRET>`.
3. The FastAPI webhook handler verifies the bearer (constant-time compare), parses `To:` to resolve the user, parses `From:` to resolve or request the source, and proceeds per the ingestion flow.

Worker code lives in `workers/inbound-email/` in this repo. Deployment via Wrangler, runbook in `deploy/`.

## Consequences

### Easier

- No mail server to run. No MX records beyond the one pointing at Cloudflare. No DKIM signing for inbound.
- Cloudflare handles TLS, rate-limiting, and SMTP compliance at the edge.
- Webhook handler is the same code path we would have written for any inbound source; trivially testable.
- Cloudflare is already a dependency of the domain. No new vendor.

### Harder

- One piece of code (the Worker) lives outside the main backend deployment. Mitigated by keeping it in-repo under `workers/` and documenting Wrangler deploy in the runbook.
- Cloudflare Email Workers have message size caps (tens of MB). Newsletters fit comfortably; very large attachments might not. Acceptable for Phase 1.
- If Cloudflare ever pulls the feature, we need to swap (to Postfix, to ForwardEmail, etc.). Abstraction keeps the cost small: the backend only cares about "raw RFC822 arrives at a webhook," not how.

### Rejected alternatives

- **IMAP polling:** added credentials and polling delay for no benefit over option 3. Also makes per-user aliases awkward.
- **Self-hosted Postfix:** operational cost of running a public-facing SMTP server is non-trivial and has no upside for a trust-group product.
- **ForwardEmail / Postmark Inbound:** viable alternatives, but introduce a new vendor. Can become a fallback if Cloudflare ever removes the Workers email feature.
