# CLAUDE.md

## ⛔ RULE #0 — STRICT TDD (MANDATORY, NO EXCEPTIONS)
1. **Write the test(s) FIRST, then the code.** Every feature. Red → green → refactor.
2. Keep tests **lean** — enough to cover all the bases, not a huge pile.
3. If code fails a test, **FIX THE CODE — never the test.**
4. **NEVER modify a test after the code is written** to make it pass, UNLESS you first ask the
   user and get **explicit permission**, with a comprehensive reason for why the test is wrong.

Silently editing a test to match buggy code defeats the entire purpose and hides regressions.
This rule outranks convenience, speed, and every other guideline in this file.

---

Guidance for working in this repo. Read this first, then the two planning docs.

## What this is
`porna` — a self-hosted adult video manager (mostly JAV ~90%, plus western + misc):
metadata scraping, a per-user library, and desktop file management.

## Authoritative docs (read before changing anything)
- **`SPEC.md`** — the SPEC (what/why). Source of truth, organized by topic. On any conflict, it wins.
- **`plan.md`** — the build ROADMAP (how/when). Milestones M1–M10; check items off as they land.
- Keep both current: when a decision changes, update `SPEC.md`; when scope/order changes, `plan.md`.

## Architecture (see SPEC.md for detail)
- Rust backend, **one role-selectable binary** (`porna run --role api,worker,webui,all`);
  modules are libraries, only `main` wires up roles.
- **Postgres** = durable state + the scrape job queue (SELECT … FOR UPDATE SKIP LOCKED).
- **Redis** = request cache + per-domain rate-limit token buckets. NOT the queue.
- **Crates:** `core` (config, pools, code normalizer, video_type classifier, filename parser,
  naming templates, shared organizer logic), `information`, `scraper`, `storage`, `api`.
- **Frontend:** one web UI; Tauri desktop (full features) / browser (browse-only, gated by
  `window.__TAURI__`). Organizer + file-watcher are desktop/app-side.
- **Auth:** external only — multiple OIDC providers or proxy forward-auth, plus per-user API
  keys; `SINGLE_USER` env flag bypasses for local/solo. No in-app passwords.

## Conventions
- Data access: `sqlx` with compile-time-checked queries. Migrations in `migrations/`,
  additive only — never edit an applied migration.
- Shared logic (code normalization, classification, filename parsing, naming templates)
  lives in `core` so backend and the Tauri app reuse it — do not duplicate it per crate.
- JAV codes are UPPERCASED before any processing/matching/storage.
- Entity/display names: title-case Latin terms; keep name history (never delete old names).
- Scraping respects per-domain rate limits; CF-bypass goes through the pluggable Fetcher
  provider (default Byparr). Never bypass the rate limiter.
- Secrets/config via env (`.env` for dev; see `.env.example`). Never commit real keys.

## Dev environment
- Local dev, **no docker**. Postgres/Redis/Byparr are **remote services** reached via env
  (`DATABASE_URL`, `REDIS_URL`, `BYPARR_URL`, etc. — see `.env.example`). Do not add docker-compose
  for datastores unless asked.

## Commands
TODO: finalize during M1 scaffolding. Expected shape:
- `sqlx migrate run`                # apply migrations to the remote DB (DATABASE_URL)
- `cargo test`                      # TDD — run constantly
- `cargo run -p api -- run --role all`
(Update this section with the real commands once the workspace exists.)
