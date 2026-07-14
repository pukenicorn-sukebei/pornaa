# porna — build plan

Living roadmap. Source of truth for requirements is **`SPEC.md`**; this file is the *how/when*.
Check items off as they land. Decisions in `SPEC.md` win on any conflict.

## stack (see SPEC.md)
- backend: Rust (axum · sqlx · reqwest/scraper · clap), one role-selectable binary
- data: Postgres (durable state + job queue), Redis (cache + rate-limit token buckets)
- assets: AssetStore trait — local FS default (`./asset/`), optional S3-compatible (DO Spaces)
- scrape CF-bypass: pluggable Fetcher provider, default Byparr (FlareSolverr-API compatible)
- frontend: one web UI; Tauri desktop (full) / browser (browse-only)
- auth: external — multiple OIDC providers or proxy forward-auth; API keys; single-user flag

## target repo layout
```
porna/
  Cargo.toml               # workspace root; members = core, backend/*, frontend/app/src-tauri
  core/                    # shared Rust: config, errors, db/redis pools, code normalizer,
                           #   video_type classifier, filename parser, naming templates,
                           #   organizer move logic. used by BOTH backend and the tauri app.
  backend/
    information/           # video + entity models, repos, name-history, external_ids, merge
    scraper/               # queue, Fetcher trait, rate limiter, source clients, scheduler
    storage/               # auth, users, locations, library, play_events, favorites, ratings, api_keys
    api/                   # axum server + clap main (role selection); serves web UI + portal
    migrations/            # sqlx migrations
  frontend/
    web/                   # shared web UI (built; served by api in browser, bundled by tauri)
    app/src-tauri/         # Tauri native shell: commands, watcher, player launch, organizer
                           #   (path-deps on core); tauri.conf.json frontendDist -> ../../web/dist
  .env.example
```

## milestones

### M1 — foundation
Local dev, NO docker; Postgres/Redis/Byparr are remote, configured via env.
- [x] CI: `.github/workflows/ci.yml` (backend + frontend jobs)
- [ ] Cargo workspace + crate skeletons (core, information, scraper, storage, api)
- [ ] config (env + file) → DATABASE_URL / REDIS_URL / BYPARR_URL etc; `.env.example`
- [ ] Postgres pool, Redis pool (connect to remote)
- [ ] AssetStore trait + local FS impl (default `./asset/`)
- [ ] sqlx migrations for the FULL schema (all modules) — runnable + inspectable
- [ ] auto-migrate on startup (api role only, pg_advisory_lock; AUTO_MIGRATE opt-out + `porna migrate`)
- [ ] `.sqlx` offline cache committed (SQLX_OFFLINE builds)
- [ ] clap `run --role ...` wiring (api/worker/webui/all), no logic yet
- [ ] /health + /ready endpoints; SIGTERM graceful shutdown (api drains; worker releases in-flight jobs)
- [ ] deploy.yml + multi-stage Dockerfile (node builds web → rust builds backend → one image
      `ghcr.io/<owner>/porna`); Tauri desktop release job (tauri-action)

### M2 — core parsing
Foundational — everything downstream depends on correct code extraction/classification.
- [ ] JAV code normalizer (uppercase) in core
- [ ] video_type classifier (jav/western/misc) in core
- [ ] filename normalization pipeline (config-driven): strip ext/[..]/(..)/quality tokens,
      detect+strip part markers (=> part_no), suffix flags (-C/-U/-4K => metadata),
      fanza-style abc00123 -> ABC-123
- [ ] naming-template token engine (incl conditional `{-part?}`) — shared with organizer
- [ ] unit tests for normalizer/classifier/parser/templates
- [ ] **tune parser against real library**: ASK USER for library path, sample filenames,
      verify parsed results WITH USER, finalize token lists

### M3 — information module
- [ ] video model: videos (+ title/title_original), video_external_ids
- [ ] entities: persons/genres/makers/labels + *_names + entity_external_ids (hybrid keying)
- [ ] junctions: video_actors, video_genres; director/maker/label FKs
- [ ] name-history helpers (add name, resolve current per locale, title-case display)
- [ ] upsert-by-(video_type, code) + pre-insert match guard
- [ ] merge admin: entities AND videos (repoint refs, consolidate, transactional, log)
- [ ] Search trait: SEARCH_MODE=trgm (pg_trgm + GIN, optional migration set) | basic (ILIKE)

### M4 — scraper skeleton
- [ ] scrape_jobs queue: enqueue / claim (FOR UPDATE SKIP LOCKED) / complete / fail+backoff
- [ ] dedup partial-unique index; priority ordering (on_demand < background)
- [ ] Fetcher trait: reqwest impl + pluggable CF-bypass (Byparr) impl
- [ ] per-domain token-bucket rate limiter (Redis)
- [ ] source clients: TPDB (REST), StashDB (GraphQL) — real; JavLibrary (HTML+CF) — stub parse
- [ ] per-type resolution chain (jav: TPDB→StashDB→JavLibrary; western: TPDB→StashDB; misc: none)
- [ ] best-of-two resolution (gap-fill, empty-only, stop after 2nd success) + completeness predicate
- [ ] match confidence: auto-accept exact/code; needs_review flag for low-confidence fuzzy
- [ ] asset download pipeline (cover/screenshots → AssetStore, keep origin url)
- [ ] date-based re-scrape rule (settle_window default 30d) + force-scrape API
- [ ] scheduler role loop

### M5 — storage module
- [ ] auth middleware: OIDC (multi-provider JWT) | proxy header | api key | single-user;
      fail-fast startup validation unless SINGLE_USER
- [ ] JIT user provisioning; admin role from env list / group claim
- [ ] locations, library_items (+ part_no, is_uncensored, ffprobe fields: resolution/bitrate/encoding/duration)
- [ ] play_events + open_count/last_opened_at (play-CLICK counter; no playback-progress tracking)
- [ ] library sync API: POST /locations/{id}/sync {added,removed,changed} (chunked, idempotent);
      app pulls→diffs→pushes; no server-side fs state; removed → cascade delete
- [ ] manual identification: assign/override video for a library_item (code or video_id),
      ignore flag, force re-scrape; "unidentified" = video_id null & not ignored
- [ ] favorites (videos, persons — separate tables), user_video_ratings
- [ ] api_keys (mint/list/revoke) + portal endpoints
- [ ] recommendations: on-demand compute + Redis cache (TTL default 24h, env)

### M6 — API surface
- [ ] REST endpoints: get video by (type, code) → enqueue on miss; browse/filter
      (genre/actor/maker/label, owned/not-owned); search; favorites; ratings; force-scrape
- [ ] serve web UI bundle + portal; pagination/filter conventions
- [ ] OpenAPI spec generation (utoipa) for the public API

### M7 — frontend
- [ ] web UI (browse/search/metadata/favorites/recs); capability gating (window.__TAURI__)
- [ ] Tauri desktop shell; secure api-key storage; native OIDC+PKCE login (or paste key)
- [ ] file-watcher (desktop) → backend library sync
- [ ] open-in-external-player (OS default + configurable command)

### M8 — organizer (desktop/app-side)
- [ ] per-type naming templates (default `{code}{-part?}.{ext}`) using the M2 token engine
- [ ] scan import → resolve → build target → move to export
- [ ] conflict: skip default + override (overwrite/suffix); manual comparison view
- [ ] manual dry-run preview + approve; scheduled live + logged (runs while app open)

### M9 — discovery + subscriptions
- [ ] subscriptions (per-user; global dedup) by person/maker/label
- [ ] new_release_discovery jobs via entity_external_ids.list_url → enqueue scrapes

### M10 — provider / integrations (this repo)
- [ ] provider API: `/provider/lookup` (metadata + image URLs), `/provider/search`;
      scrape-on-miss → 202; authed via api_keys
- [ ] publish OpenAPI spec for the provider API (external consumers build against the contract)
- note: the **Jellyfin plugin is a SEPARATE repo** (C#/.NET, own CI/release, codegen from OpenAPI);
  nothing shared, so it's out of this monorepo's build scope

### M11 — polish / optional
- [ ] S3-compatible AssetStore impl (DO Spaces)
- [ ] genre normalization review workflow (hybrid keying + merge)
- [ ] merge admin UI
- [ ] in-app playback (low priority)

## cross-cutting
- **STRICT TDD (see CLAUDE.md RULE #0): test first, then code; never edit a test to pass code
  without explicit permission.** Every checklist item = write its test(s) first.
- logging/tracing, error taxonomy in core
- test strategy: unit (normalizer/classifier/templates), integration (sqlx queries, queue)
- migrations are additive; never edit an applied migration

## open items / assumptions
- toolchain: local vs docker for PG/Redis/Byparr (PENDING)
- library path for filename tuning (ASK during M2)
- scraper per-site parse specifics (user to provide as sources are implemented)
