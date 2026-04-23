# Sieve: project instructions

## Current status

Sieve is a concept project. The repo currently contains design documents only:

- [Design spec](../docs/superpowers/specs/2026-04-23-sieve-platform-design.md)
- [OPEN_THINGS](../docs/superpowers/OPEN_THINGS.md) (work catalog)
- [Architecture docs](../docs/architecture/) and [ADRs](../docs/adr/)

No backend, frontend, or worker code has been written yet. Phase 0 (see OPEN_THINGS) will land the tooling and scaffold. Anything in this file that references build commands, containers, or test runners describes the target state, not what exists today.

## Where rules live

- **This file (`.claude/CLAUDE.md`)**: project-wide architecture, workflow, conventions. Loaded every session.
- **`backend/CLAUDE.md`** (after Phase 0): Python / FastAPI / SQLAlchemy / pytest rules scoped to backend files.
- **`frontend/CLAUDE.md`** (after Phase 0): React / Vite / Biome / Vitest rules scoped to frontend files.
- **`workers/inbound-email/CLAUDE.md`** (after Phase 0): Cloudflare Worker rules scoped to the worker.
- **`.claude/rules/*.md`** (as needed): rules scoped by file path via `paths:` frontmatter.

See [Anthropic's memory docs](https://code.claude.com/docs/en/memory) for the loading model.

## Architecture (target state)

- **`backend/`**: FastAPI + SQLAlchemy async web container. REST API, magic-link auth, webhook endpoints. Runtime state (engine, session factory, settings) on `app.state` set in `lifespan`.
- **`backend/`** also hosts the **worker** entry point: APScheduler + DB-backed job queue (`SELECT ... FOR UPDATE SKIP LOCKED`). Packaged as the same Python module, run as a separate container with a different command.
- **`frontend/`**: Vite + React + TypeScript SPA. Proxies to backend in dev. OpenAPI-codegen types (`mise run fe:types`).
- **`workers/inbound-email/`**: Cloudflare Email Worker. JS, deployed via Wrangler. POSTs raw RFC822 to the backend's webhook.
- **`deploy/`**: compose files for staging + production. GHCR images published by GitHub Actions on master. Joins the external `web` network run out of `~/Code/server-infra/`.
- **Datastores:** Postgres 17 with pgvector (relational + vectors + job queue), MinIO (content and MIME blobs).
- **LLM:** Anthropic + Google Gemini via pluggable `Summarizer` / `Embedder` interfaces. BYOK per user, platform key for paid users.

Full picture in [`docs/architecture/overview.md`](../docs/architecture/overview.md).

## Development workflow

Sieve follows the same flow as Klassenzeit.

**Skills are not optional when a workflow names them.** Slash commands and skill references (notably the ones in this file) call out specific skills by name. "Invoke the skill" means call the `Skill` tool and let it return, then follow what it says. Synthesizing a skill's output freehand, even when it looks right, is skipping the skill and counts as a process violation. If a workflow step names a skill, calling `Skill` is the first action of that step.

Always use TDD with red-green-refactor, driven by `superpowers:test-driven-development`. Development always ends in PRs after docs and ADRs have been reviewed and updated.

Before opening a PR, run `claude-md-management:revise-claude-md` if the session produced learnings worth persisting, and if you ran that, run `claude-md-management:claude-md-improver` right after. Both via the `Skill` tool.

Keep things that are out of scope for a step, or that you notice as tech debt or todos, in [`docs/superpowers/OPEN_THINGS.md`](../docs/superpowers/OPEN_THINGS.md), ordered by importance. Don't add duplicates.

## Work selection: quality first, tidy first

When picking the next item off OPEN_THINGS.md without a more specific directive from the user, prefer **tech debt and quality work over new user-facing features**. Follow Kent Beck's "Tidy First?" heuristic: small structural refactors that make subsequent feature work cheaper and safer come before the features themselves. Concretely:

1. Read OPEN_THINGS.md top to bottom. Skim the "Product capabilities" section last.
2. Pick the highest-impact item from "Infrastructure", "Testing", "Observability", "Open decisions", or similar quality-adjacent sections that is unblocked and fits a single PR.
3. Structural refactors that remove duplication, collapse drift between near-identical call sites, or replace ad-hoc patterns with shared primitives count as tidy-first and are preferred over feature work.
4. A structural change and a behavioral change never ship in the same commit. If a tidy-first refactor uncovers a behavior bug, surface the bug and fix it in a separate commit with its own typed prefix (`fix(...)`), not folded into the refactor.
5. Behavior must be preserved across a tidy commit: tests that passed before must pass after without modification, except where the only change is a test's import path or a mock signature rendered obsolete by the refactor.

If every quality item in OPEN_THINGS.md is blocked or out of scope for one PR, fall back to the next feature item and note why in the PR body.

## Tooling (target state after Phase 0)

### Commands

- `mise run dev`: backend with auto-reload
- `mise run fe:dev`: frontend dev server
- `mise run worker`: worker container locally
- `mise run test`: all tests
- `mise run test:py`: Python only
- `mise run fe:test`: frontend Vitest only
- `mise run e2e`: Playwright suite
- `mise run lint`: all linters
- `mise run fmt`: auto-format
- `mise run fe:types`: regenerate frontend OpenAPI types from backend
- `mise run db:up` / `db:stop` / `db:reset` / `db:migrate`: Postgres lifecycle

### Prerequisites

- **mise** pins Python, Node, uv, pnpm, cocogitto, lefthook. `mise install` on fresh clones.
- **Git hook runner:** [Lefthook](https://github.com/evilmartians/lefthook). Config at `.config/lefthook.yaml`.
- **Commit-message enforcement:** [Cocogitto](https://docs.cocogitto.io) (`cog`). `commit-msg` hook runs `cog verify`.
- **Pre-push runs the full test suite.** Even docs-only pushes pay the cost. Broken builds never reach the remote. Use `mise exec -- git push` so the pinned lefthook runs.
- **`gh` + `jq`** required for repo automation tasks.

## Coding standards

- **No bare catchalls.** No untyped `catch` in TypeScript, no `Result<_, _>` swallowed with `_` in Rust (if ever added), no bare `except:` in Python. Catch the specific error you can handle; let the rest propagate.
- **No dynamic imports.** All imports static / top-of-file so the dependency graph is analyzable. No `import()` expressions, no `importlib.import_module` in hot paths.
- **Unique function names globally.** Function names must be unique across the entire codebase, even across classes and files. A pre-commit check enforces this once Phase 0 lands the script.
- **ADR titles skip the em-dash.** Use a colon: `# NNNN: Title`. The user's global style forbids em-dashes and en-dashes in prose.
- **SHA-pin third-party GitHub Actions.** `actions/*` and `github/*` can use `@vN`; everything else pins to a full commit SHA with a trailing `# vX.Y.Z` comment.
- **Conventional Commits.** `<type>(<scope>)?: <subject>`. Subject starts lowercase. Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`. Append `!` for breaking changes. PR titles follow the same rule.

## ADR-first development

All architectural decisions are documented in [`docs/adr/`](../docs/adr/). The ADRs are the source of truth.

Before implementing any new feature, structural change, or pattern deviation:

1. **Check ADRs**: read relevant ADRs to understand existing decisions.
2. **Update or create ADR**: if the change contradicts an existing ADR, update it (set old to "Superseded"). If the change introduces a new pattern, create a new ADR with status "Proposed".
3. **Implement**: follow the ADR.
4. **Accept the ADR**: once the feature ships and the decision has held in practice, flip `Status: Proposed` to `Status: Accepted`.

Nothing gets implemented without being documented in an ADR first, or pointing at the ADR that already covers it.

## Commit messages

This repo will enforce [Conventional Commits](https://www.conventionalcommits.org/) once Phase 0 lands `cog`.

**Format:** `<type>(<optional scope>): <description>`

**PR titles** must also satisfy `subjectPattern: ^[a-z].+$`. Start the subject with a lowercase letter even when the first word is an acronym: `feat(frontend): crud pages ...`, not `feat(frontend): CRUD pages ...`.

**Never add AI attribution** to commits, PRs, or code. No "Generated with", no "Co-Authored-By: Claude".

## Writing style (enforced via user global, repeated here for clarity)

- No em-dashes (—) or en-dashes (–) in prose. Rewrite with commas, periods, colons, semicolons, parentheses, or separate sentences.
- No hyphens (-) as sentence punctuation (" - " standing in for a comma or dash). Hyphens in compound words are fine.
- Markdown horizontal rules (`---`) are structural, not punctuation.

Applies to all prose: commit messages, PR bodies, docs, code comments.
