# ADR 0004: BYOK LLM keys with optional platform key for paid users

**Status:** Proposed
**Date:** 2026-04-23

## Context

Sieve uses LLM calls for summarization (Phase 1) and embeddings + re-ranking + tagging (later phases). Providers in scope: Anthropic and Google Gemini. These are paid APIs; at any real usage, they represent the bulk of running cost.

Three models exist for who pays:

1. **Platform absorbs cost.** Operator pays for all LLM usage. Simple UX; cost is unbounded unless rate-limited.
2. **BYOK only.** Every user must bring their own Anthropic/Google key. Cost is clean but friction is high; users who do not have or want an API key are blocked.
3. **BYOK with platform key as an option.** Users can bring a key (preferred default), or opt into a paid plan that uses the platform's key with a per-user budget cap.

The trust group includes users of varying technical willingness to manage API keys. The operator also wants attribution of cost and a mechanism to bill a user for heavier usage.

## Decision

BYOK is the default; the platform key is a paid-plan convenience.

Resolution at each LLM call site:

```
resolve_key(user, provider):
  if user has BYOK for provider:
      return user's BYOK (decrypted)
  elif user.plan == 'paid':
      return platform key
  else:
      raise NoKeyAvailable
```

All LLM calls write a row to `llm_usage` with `tokens_in`, `tokens_out`, and `cost_cents`, keyed by `(user_id, period, provider)`. Platform-key usage is capped by a configurable soft cap per user per month.

BYOK keys are stored encrypted at rest with `cryptography.Fernet` using a server-side master key from env. Platform keys live only in env.

Payment is handled out-of-band in Phase 1: the operator flips `users.plan` directly in the database. A proper payment flow is future work and does not require architecture change.

## Consequences

### Easier

- Cost is always attributable to a user; no runaway platform bill is possible without visibility.
- Users with their own keys (the default) are unblocked without any paid infrastructure on our side.
- Swapping in a local LLM later (Ollama, llama.cpp) is an additional adapter, not an architectural change; `resolve_key` returns a provider-specific credential, and the adapter decides what to do with it.

### Harder

- Two code paths for key resolution (BYOK vs platform). Encapsulated in one function; acceptable.
- Platform-key budget capping requires tracking usage in near-real-time. `llm_usage` is updated inside the worker at call completion; cap enforcement is a cheap SELECT before each call.
- Summaries and embeddings are shared across subscribed users (they are per-item, not per-user). The first ingesting user's key pays for the item's summary and embedding; other users benefit. This is a known asymmetry and is explicitly acknowledged; it is acceptable because the per-item cost is small and the asymmetry is small on average.

### Rejected alternatives

- **Platform absorbs all cost:** open-ended cost exposure for the operator; not viable without immediate billing.
- **BYOK only, no platform key:** high friction for non-technical users in the trust group.
