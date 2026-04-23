# Security

## Threat model at a glance

Sieve is not a public service. The trust model assumes:

- Users are known to the operator and vetted at invite time.
- The operator is not hostile to their own users.
- The VPS is behind Hetzner's network protections.
- The primary adversaries are: internet-scale opportunistic attackers, compromised upstreams (e.g. a hijacked RSS feed), and misconfigured defaults.

The platform does NOT attempt to defend against a malicious operator (that is outside the threat model for a personal/trust-group tool).

## Principles

1. **Defense in depth over single chokepoints.** Auth, transport, database, and application all enforce their own constraints.
2. **No secrets in code, logs, or source control.** Every secret comes from env at process start.
3. **Least privilege at every boundary.** DB user has only what it needs. Webhook endpoints are individually authenticated. API tokens (Phase 3) are scoped.
4. **Fail closed.** Missing auth, missing keys, and decryption failures always produce errors, never fallbacks.

## Auth

### Login

Magic-link email login. No passwords. Flow:

1. User enters email.
2. Server mints a signed, short-lived (15 min) token and emails it as a link.
3. User clicks, token is verified, session cookie is set.

Tokens are signed with `itsdangerous` or equivalent and include a usage nonce to prevent replay.

### Sessions

Server-side session cookie with `HttpOnly`, `Secure`, `SameSite=Lax`. Default session length 30 days. Session lookup is a Postgres row, not a signed cookie, so revocation is immediate.

### Invite-only signup

No `POST /signup` endpoint. Instead, `POST /redeem?code=...` creates an account against an issued invite code. Invite codes have expiries and single-use flags.

### Admin

The operator is a user flagged `is_admin`. Admin endpoints are gated by a dependency check; no separate admin subsystem. Admin UI is the same SPA with conditional routes.

## Secrets

### BYOK API keys

Encrypted at rest with `cryptography.Fernet` using a server-side master key loaded from env (`SIEVE_SECRETS_MASTER_KEY`). Decrypted at LLM call time, never cached on disk, never logged. The master key is never rotated in Phase 1; rotation is a Phase 2 concern and requires a re-encryption migration.

### Platform LLM keys

Live in env (`ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`). Used only when `resolve_key(user, provider)` falls through to platform. Never written to the database.

### Cloudflare inbound webhook secret

`SIEVE_INBOUND_SECRET` in env on both the web container and the Cloudflare Email Worker's environment. Compared with `secrets.compare_digest` (constant-time).

### Session signing key

`SIEVE_SESSION_KEY` in env. Rotation invalidates all sessions; acceptable cost.

## Transport

- HTTPS only via Caddy.
- HSTS with a long max-age after Phase 0 shakedown.
- No HTTP fallback, no mixed content.
- Strict CSP: `default-src 'self'`, `script-src 'self'`, no inline scripts, no external script sources. Adjusted per-page if a specific integration needs it.

## Input validation

- All API inputs go through Pydantic models. No loose dict parsing.
- HTML content from RSS and email is sanitized with `bleach` or `nh3` before being served in the reader. XSS via feed content is a real threat.
- URLs in feeds are not automatically fetched by the web container; only the worker fetches upstream URLs.

## Database

- DB user for the app has `CRUD` on sieve tables only; no `DROP`, no `CREATE`, no superuser. Migrations run under a separate migration role.
- All queries via SQLAlchemy with parameterization. No f-strings into SQL, ever.
- `UNIQUE` and `FOREIGN KEY` constraints on all subscription and item relationships. Access checks are enforced in code, not relied on at the DB layer; the DB layer is the second line of defense.

## Rate limiting

- Auth endpoints (`/auth/magic-link`, `/auth/redeem`): per-IP and per-email burst + sustained limits.
- Inbound webhook (`/webhooks/inbound-email`): per-IP limit as a belt-and-braces measure behind the shared secret.
- API endpoints (Phase 3): per-user token limit.

Implementation in Phase 1 via `slowapi` or equivalent; Redis is not required at Phase 1 scale, but the interface leaves room to swap in a shared store later.

## File and blob safety

- Objects in MinIO are addressed by opaque UUID keys, never by user-controlled names.
- HTML content blobs are always sanitized on read, never at write time, so the raw original is preserved for re-sanitization if the sanitizer is updated.
- Raw MIME blobs are read-only after write.

## Email-specific

- Inbound: DKIM/SPF validation is performed upstream by Cloudflare; we log the results but do not hard-reject on SPF-fail (some legitimate forwarded newsletters fail SPF).
- Outbound: all transactional email goes through a single provider (Resend or Postmark) with verified DKIM. Bounce and complaint webhooks are wired to mark emails as undeliverable.

## Audit

- All admin actions (approve source, reject source, flip `plan`, invite, delete) write to a simple `audit_log` table with `actor_id`, `action`, `target`, `payload_json`, `created_at`.
- Webhook deliveries are logged with `trace_id` and outcome.

## Legal / content

The subscription-gating rules in the top-level design doc (Principles #2 and #3) are the primary copyright-risk mitigation. Implementation MUST:

- Reject any inbound email from a sender not canonicalized as a source that has at least one subscribed user.
- Refuse to return items for a user who does not have an active subscription to the item's source, even by item id lookup.
- Provide a delete-my-data flow on user request.

## Secrets bootstrap (Phase 0 checklist)

- [ ] `SIEVE_SECRETS_MASTER_KEY` generated and stored in env
- [ ] `SIEVE_SESSION_KEY` generated and stored in env
- [ ] `SIEVE_INBOUND_SECRET` generated and stored in env + Cloudflare Worker env
- [ ] `ANTHROPIC_API_KEY` / `GOOGLE_API_KEY` (platform keys) set
- [ ] Transactional email provider API key set
- [ ] Postgres credentials set, migration role separate from app role
- [ ] MinIO root and app credentials set, app key scoped to the sieve bucket only

None of the above keys ever appear in this repo or in logs.
