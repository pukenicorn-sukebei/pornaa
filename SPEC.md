# porna — specification

Source of truth for **what** we're building and **why**. Build order/progress lives in `plan.md`;
working conventions in `CLAUDE.md`. On any conflict, this file wins.

## 1. Overview
A self-hosted **adult video manager**. Content is mostly JAV (~90%), plus western and some misc.
It scrapes and stores video metadata, tracks a per-user library of owned files, and provides a
desktop app for browsing, playback, file-watching, and file organization.

Three backend modules — **scraper**, **information**, **storage** — built as a modular monolith
(one repo, one role-selectable binary; splittable into services later).

## 2. Stack
- **Backend:** Rust (async/tokio). axum (HTTP), sqlx (compile-time-checked queries, migrations
  via sqlx-cli), reqwest + scraper (scraping), clap (CLI/role selection).
- **Postgres:** durable state **and** the scrape job queue. Committed for v1 — portability to other
  SQL (MySQL/MariaDB) is a **non-goal**; the schema leans on Postgres-specific features (`text[]`
  arrays, `jsonb`, `FOR UPDATE SKIP LOCKED`). Keep the data-access layer tidy, but don't contort the
  schema chasing portability.
- **Redis:** request/response caching (metadata lookups, API responses, recommendations, browse)
  and per-domain rate-limiter token buckets. **Not** the queue.
- **Assets:** `AssetStore` trait — local FS default (`./asset/`, env-configurable base dir),
  optional S3-compatible (e.g. DigitalOcean Spaces).
- **Frontend:** Vite + Svelte **static SPA** (no SSR — the same static `dist/` bundles into Tauri
  and is served by the Rust backend for the browser). `ts-rs` generates TS types from Rust API
  structs. One web UI; Tauri desktop (full) / browser (browse-only). See §8.
- **Dev:** local, no docker. Postgres/Redis/Byparr are remote, configured via env.
- **Config — connection strings:** each datastore accepts a full `*_URL` **or** individual
  components. The `*_URL` **takes precedence** when set; otherwise the connection is **built from
  components**. Test connections **inherit the workload components as defaults** (override with
  `TEST_*`), so you configure the server once.
  - Postgres workload: `DATABASE_URL` | `PG_HOST` `PG_PORT`(5432) `PG_USER` `PG_PASSWORD`
    `PG_DATABASE` `PG_SSLMODE`
  - Postgres test: `TEST_DATABASE_URL` | `TEST_PG_*` — each `TEST_PG_*` defaults to the matching
    `PG_*`, **except the database, which defaults to `pornaa-test`** (override via `TEST_PG_DATABASE`)
  - Redis workload: `REDIS_URL` | `REDIS_HOST` `REDIS_PORT`(6379) `REDIS_USERNAME` `REDIS_PASSWORD`
    `REDIS_TLS`(on/off) `REDIS_DB`(number)
  - Redis test: `TEST_REDIS_URL` | `TEST_REDIS_*` — defaults to the matching `REDIS_*`, except
    `REDIS_DB` which defaults to `1` (keeps test cache off the workload DB)

## 3. Runtime — single role-selectable binary
One binary; a clap CLI selects which components run in-process; all share the same config.
Modules are libraries — only `main` wires up roles.
```
porna run --role all
porna run --role api
porna run --role worker
porna run --role webui
porna run --role api,worker      # any combination
```
- **api** — HTTP API (information + storage); also serves the web UI bundle + portal.
- **worker** — scraper queue consumers + schedulers (discovery, re-scrape; optionally headless
  organizer scheduling).
- **webui** — serve the web UI bundle (can be folded into `api`, or split to scale separately).

Run everything on one box now; split roles across processes/machines later with no code change.

### migrations on startup
- Migrations are embedded in the binary and **applied automatically on startup by the `api` role
  only** (worker/webui don't migrate).
- **Race-safe across multiple `api` replicas:** sqlx's Postgres migrator takes a `pg_advisory_lock`
  before applying — the first instance migrates, the others block then find nothing pending.
- Non-`api` roles **verify** the schema is at/above the expected version and fail-fast if they
  started before `api` migrated (so start-order bugs surface loudly).
- Escape hatch: `AUTO_MIGRATE=false` env + a `porna migrate` subcommand for ops who prefer a
  separate migration step.

## 4. Core parsing (crate `core`, shared by backend + Tauri app)
- **JAV code normalization:** JAV codes are UPPERCASED (and trimmed/dash-normalized) before any
  processing, matching, or storage. `abc-123` → `ABC-123`.
- **video_type classification** (implicit from a raw code/query): ordered, rule-based regexes
  with confidence; manual override allowed.
  - jav: `ABC-123`, `ABCD-00123`, label+dash+digits, fanza content-id style (`118abc00123`)
  - western: studio/scene title strings, long numeric scene ids, non-jav url patterns
  - default to `misc` when nothing matches confidently
- **Filename normalization pipeline** (config-driven token lists):
  strip extension → strip `[..]`/`(..)` junk → strip quality tokens (`1080p`, `x264`, `WEB-DL`, …)
  → detect+strip part markers (`cd1`, `part2`, `-A`/`-B`) ⇒ `part_no`
  → detect+strip suffix flags (`-C` subs / `-U` uncensored / `-4K`) ⇒ metadata
  → normalize fanza-style `abc00123` → `ABC-123` → hand cleaned code to the classifier.
  Note: tune against the user's real library filenames and verify results with them before locking.
- **Naming-template engine** (shared with organizer): tokens `{code} {title} {maker} {label}
  {actor} {studio} {year} {part} {ext} {video_type}`; conditional `{-part?}` renders `-{part}` only
  if a part number is present.
- **Display-name casing:** title-case each space-separated word (capitalize first char, keep rest);
  applied to Latin/English terms, no-op/safe on CJK; guard empty strings / multi-space.

## 5. Information module — data model
Conventions: UUID surrogate PKs. Named entities (person, genre, maker, label) never store their
name inline — names live in a per-entity `*_names` table. "Current" name = latest added per
(entity, locale): `ORDER BY is_primary DESC, created_at DESC` (`is_primary` is an optional manual
pin). All name rows are kept (history); no strict chronology required. Locales `en`/`ja` to start,
arbitrary codes allowed. maker/label are English-only; actor/director/genre are multilingual.

### videos
```
id            uuid pk
code          text not null              -- natural id; JAV uppercased
video_type    text not null              -- 'jav' | 'western' | 'misc'
title         text                       -- display, usually EN/romaji
title_original text                      -- source language, usually JA
length_min    int  check (length_min > 0 and length_min <= 1000)   -- minutes
release_date  date
cover         text                       -- local asset ref (downloaded, see §6)
screenshots   text[] not null default '{}'  -- local asset refs
director_id   uuid  -> persons(id)        -- single director
maker_id      uuid  -> makers(id)
label_id      uuid  -> labels(id)
primary_source text                       -- optional pointer; per-source ids in video_external_ids
needs_review  bool not null default false -- low-confidence (fuzzy) match flagged for manual confirm
scraped_at    timestamptz
created_at / updated_at timestamptz
UNIQUE (video_type, code)
```
Censored/uncensored is **not** a separate video — one row per code; the distinction is per-file on
`library_items.is_uncensored` (§7).

**Match confidence / review:** exact code matches (and direct API id hits) are auto-accepted;
low-confidence fuzzy matches (western: title+date+studio) set `needs_review=true` so they surface for
manual confirm/reject instead of being trusted blindly.

### video_external_ids  (per-source identity — drives re-scrape + cross-source matching)
```
id, video_id -> videos(id) on delete cascade,
source, external_id, url, fetched_at,
UNIQUE (source, external_id)
```
One video may have rows for TPDB + StashDB + JavLibrary simultaneously.

### persons (shared by actor + director)
```
persons: id uuid pk, created_at
person_names: id, person_id -> persons on delete cascade, locale, name,
              is_primary bool default false, created_at, UNIQUE (person_id, locale, name)
video_actors (m-m): video_id -> videos on delete cascade, person_id -> persons,
                    PK (video_id, person_id)
director: videos.director_id FK (single) — same persons table
```

### genres
```
genres: id uuid pk, created_at
genre_names: genre_id -> genres, locale, name, is_primary, created_at,
             UNIQUE (genre_id, locale, name)
video_genres (m-m): video_id -> videos on delete cascade, genre_id -> genres,
                    PK (video_id, genre_id)
```

### makers / labels (English-only; western studio→maker, network/series→label)
```
makers: id uuid pk, created_at
maker_names: maker_id -> makers, locale('en'), name, is_primary, created_at, UNIQUE (maker_id, name)
labels: id uuid pk, created_at
label_names: label_id -> labels, locale('en'), name, is_primary, created_at, UNIQUE (label_id, name)
videos.maker_id / videos.label_id FKs (single each)
```
No separate `series` entity: JAV has no series concept here; western series/network folds into label.

### entity_external_ids  (per-source ids for entities — discovery + matching; hybrid keying)
```
id, entity_type IN ('person','maker','label','genre'), entity_id,
source, external_id, source_term, list_url, fetched_at,
UNIQUE (entity_type, source, external_id)
```
- `external_id` = the source's real id if it has one, else the NORMALIZED (lowercase, trimmed)
  term — so the UNIQUE key works with or without a source id.
- `source_term` = raw human-readable term (display/review).
- `list_url` = page listing this entity's videos (feeds new-release discovery).
- Lookup on scrape: match by real id, else by normalized term; miss → create entity + mapping row.

### merge (dedup admin; entities AND videos)
- **Entities** (person/genre/maker/label): pick survivor id, repoint all refs
  (video_actors.person_id, videos.director_id/maker_id/label_id, video_genres.genre_id),
  consolidate `*_names` (dedup identical (locale,name), keep all), transactional, log merged-from.
- **Videos**: pick survivor, repoint library_items / favorite_videos / play_events / video_actors /
  video_genres / video_external_ids / scrape_jobs / user_video_ratings; union relations, keep both
  external-id rows, fill empty survivor fields from loser; transactional, log merged-from.
- **Pre-insert match guard:** check `video_external_ids` and `(video_type, code)` before creating a
  video, to avoid duplicates in the first place.

### search (opt-out backend)
Free-text search over codes, titles, and entity names sits behind a small `Search` trait, selected
by `SEARCH_MODE` env:
- **`trgm` (default):** `pg_trgm` extension + GIN trigram indexes → fuzzy/substring/typo-tolerant.
  The extension + indexes live in a **separate, optional migration set** applied only when this mode
  is on (run by the `api` role at startup alongside core migrations, same advisory-lock).
- **`basic` (opt-out):** no extension, no trigram indexes — `ILIKE` prefix/substring fallback. Works
  on locked-down managed Postgres that disallows `CREATE EXTENSION`; slower on large libraries.
Facet browse (by genre/actor/maker/label) is plain indexed SQL and works in both modes.

## 6. Scraper module
Runs as a queue consumer/worker under the `worker` role.

### scrape_jobs (Postgres-backed durable queue)
```
id bigserial pk
job_type   text     -- 'on_demand' | 'rescrape' | 'new_release_discovery'
video_type text, code text     -- target (null for discovery)
site       text
payload    jsonb              -- extra params (e.g. entity id for discovery)
priority   int                -- lower = higher; on_demand < background
status     text               -- 'pending' | 'in_progress' | 'done' | 'failed' | 'dead'
attempts   int default 0, max_attempts int
run_after  timestamptz        -- backoff scheduling
last_error text
created_at / updated_at timestamptz
dedup: partial UNIQUE index on (video_type, code, job_type) WHERE status IN ('pending','in_progress')
claim: WHERE status='pending' AND run_after<=now()
       ORDER BY priority, created_at FOR UPDATE SKIP LOCKED LIMIT 1
```
Retries with exponential backoff.

### fetcher
`Fetcher` trait: (a) plain reqwest; (b) CF-bypass — a **pluggable provider** chosen by env,
default **Byparr** (Camoufox-based, speaks the FlareSolverr API so client code is provider-agnostic;
swappable to FlareSolverr, or escalate to a solver API / real browser if a site goes full Turnstile).
Nothing deployed yet → config points at a remote Byparr endpoint. Per-site config picks fetcher +
rate limit; CF-bypass path gets a tighter limit.

### sources & resolution chain (ordered per video_type; best-of-two)
```
jav:     [TPDB, StashDB, JavLibrary]
western: [TPDB, StashDB]
misc:    []                          -- no auto-scrape; manual entry only
```
Lists are config-driven.

**Best-of-two resolution:** walk the chain in order.
- **Success #1** populates the record. If it is now **complete**, stop (call no one else).
- If still incomplete, continue to the next source that returns data → **success #2** → gap-fill
  (**empty fields only**, so #1 keeps per-field precedence).
- **Stop after the second successful source**, regardless of completeness — never consult a third.
- Misses don't count toward the cap; only successful fetches do.
- **Complete** = a configurable predicate over required fields (e.g. title, cover, actors,
  release_date, genres, maker/label).
- **TPDB** (ThePornDB REST, bearer token) — primary, broadest coverage.
  env `TPDB_API_KEY` (required to enable), optional `TPDB_ENDPOINT`.
- **StashDB** (GraphQL `https://stashdb.org/graphql`). env `STASHDB_API_KEY` (required),
  optional `STASHDB_ENDPOINT`. Rate limit 2 req/s, ≥0.5s spacing.
- **JavLibrary** (jav only; HTML scrape, clean data, behind Cloudflare → CF-bypass fetcher).
- Each source: conservative configurable per-domain rate limit (Redis token bucket).
- Candidates for later: JavBus, JavDB, DMM/FANZA, AVBase/xslist.

### media assets
Covers/screenshots are **downloaded and cached** (not hotlinked) via `AssetStore` (local default
`./asset/`, optional S3). Keep the origin URL (in `video_external_ids` or an asset origin field) for
re-fetch. Asset fetches obey the same rate limit / CF-bypass as scraping.

### freshness / re-scrape (date-based)
```
needs_rescrape = release_date IS NOT NULL
             AND now() >= release_date + settle_window
             AND fetched_at < release_date + settle_window
```
Exactly one self-limiting catch-up re-scrape after the settle window (default 30d, env-configurable).
Null release_date → no auto re-scrape. A **force-scrape API** enqueues a high-priority re-scrape on
demand.

### triggers
(a) requested code missing → on-demand scrape; (b) info stale (rule above) → re-scrape;
(c) scheduled new-release discovery by actor/maker/label → background scrapes.

### new-release discovery + subscriptions
```
subscriptions: id, user_id -> users, target_type IN ('person','maker','label'), target_id,
               cadence, last_run_at, enabled, created_at        -- per-user; discovery deduped globally
```
Scheduler: due subscription → look up the entity's per-source `list_url` (entity_external_ids) →
enqueue `new_release_discovery` → new codes become scrape jobs.

## 7. Storage module
The user-facing module the frontend talks to.

### auth (external; backend is auth-method-agnostic)
No in-app passwords. Identity comes from:
- **multiple OIDC providers** — a configurable list (issuer + client id/secret + JWKS); validate a
  bearer JWT against any configured issuer.
- **proxy forward-auth** — a configurable trusted header (e.g. `Remote-User`/`X-Forwarded-User`)
  injected by Authelia / Authentik / oauth2-proxy.

**Startup validation:** if neither any OIDC provider nor proxy auth is configured, **fail fast** —
unless an explicit single-user flag (`SINGLE_USER=true`) trusts a configured default subject
(local/solo). Users are JIT-provisioned on first authenticated request. Admin role from a
configurable admin-subject list (env) or an OIDC/proxy group claim.

Per surface: web = OIDC or proxy; desktop = native OIDC Authorization Code + PKCE (RFC 8252) via
system browser, or an API key; headless agents (file-watcher/CLI) = API keys. The backend resolves
identity uniformly from any of: OIDC JWT | proxy header | API key | single-user.

### tables
```
users: id uuid pk, issuer, subject, username, email, role default 'user' ('user'|'admin'),
       created_at/updated_at, UNIQUE (issuer, subject)

locations: id uuid pk, user_id -> users on delete cascade, name, root_path, created_at,
           UNIQUE (user_id, name)

library_items: id uuid pk, user_id -> users on delete cascade, location_id -> locations on delete cascade,
               video_id -> videos (null until matched), raw_code, video_type, file_path, file_size,
               part_no int,                         -- multi-part; several items may share one video_id
               is_uncensored bool,                  -- per-file (from filename); censored/uncensored copy
               resolution text,                     -- e.g. '1080p','2160p'  (ffprobe, cached)
               bitrate int,                         -- kbps  (ffprobe, cached)
               encoding text,                       -- codec: h264/h265/av1  (ffprobe, cached)
               duration int,                        -- seconds (file duration; distinct from videos.length_min)
               present bool default true, added_at/last_seen_at,
               open_count int default 0, last_opened_at,   -- denormalized from play_events
               UNIQUE (user_id, location_id, file_path)
  -- file-technical fields populated by ffprobe at scan (cached, refreshable on demand),
  --   not recomputed per query, so browse/sort/organizer-compare work even when the file is offline
  -- open_count/last_opened aggregate per (user, video_id) for display
  -- no content_hash in v1 (code-based identity + cheap db lookup handles moves)

play_events: id bigserial pk, user_id -> users on delete cascade,
             library_item_id -> library_items on delete cascade, video_id -> videos,
             opened_at timestamptz default now()
  -- on insert: bump library_items.open_count + last_opened_at

favorite_videos:  user_id -> users, video_id -> videos, created_at, PK (user_id, video_id)
favorite_persons: user_id -> users, person_id -> persons, created_at, PK (user_id, person_id)

user_video_ratings: user_id -> users, video_id -> videos, rating, updated_at,
                    PK (user_id, video_id)      -- can rate not-owned videos too

api_keys: id uuid pk, user_id -> users on delete cascade, name, key_hash, prefix,
          created_at/last_used_at, expires_at, revoked bool default false
```

### recommendations ("video / actor of the day")
Not persisted in Postgres — computed on demand + cached in Redis under a per-user key, regenerated
when expired. TTL default 24h, env-configurable (`REC_TTL_SECONDS`).

### file-watch flow (frontend → backend)
Watcher finds file → parse code from filename → classify video_type → ensure `videos` row exists
(enqueue scrape on miss) → upsert `library_item`. Opening a file → insert `play_event` (+ counters).

## 8. Frontend
Built as a **Vite + Svelte static SPA** (no SSR; there is no Node server in production — the Rust
backend serves the static bundle). Rationale: the same `dist/` must both bundle into Tauri and be
served in the browser as static files, which rules out SSR-first frameworks (Next/Nuxt/SvelteKit-SSR/
Qwik). `ts-rs` generates TS types from the Rust API structs to keep the contract in sync.

One web UI codebase, two runtimes; capability-gated at runtime (e.g. `window.__TAURI__` present):
- **Desktop (Tauri):** cross-platform Win/macOS/Linux from one codebase. Full features —
  location management, filesystem watching, playback / open-in-external-player, organizer.
- **Browser:** same bundle served over the web (behind OIDC/proxy). Browse / search / metadata /
  favorites / recommendations only; location selection + playback hidden/disabled (no fs / native
  player).
- **Mobile:** out of scope (file-watch + external player are desktop concepts).

Features: browse collections by genre/actor/maker/label (owned or not owned); random recommended
videos; open in external player (OS default association out of the box + optional configurable player
command — path + per-OS arg template, e.g. `mpv "{path}"` — in Tauri local settings); in-app playback
is low-priority/optional.

### api-key portal
The served web UI **is** the portal: browser users authenticate via OIDC/proxy, browse, and
generate/revoke API keys there. Desktop app either logs in via native OIDC+PKCE or the user pastes an
API key once (stored in secure local storage). Single-user mode needs no portal.

### repo layout
Single monorepo (shared API request/response types kept in sync). `core/` is its own top-level dir
because both the backend and the Tauri app depend on it.
```
porna/
  Cargo.toml               # workspace root; members = core, backend/*, frontend/app/src-tauri
  core/                    # shared Rust: config, pools, code normalizer, classifier,
                           #   filename parser, naming templates, organizer move logic
  backend/
    information/  scraper/  storage/  api/   # api = the clap binary (role selection)
    migrations/                              # sqlx migrations
  frontend/
    web/                   # shared web UI — served by api in browser, bundled by tauri
    app/src-tauri/         # Tauri native shell (commands, watcher, player, organizer);
                           #   path-deps on core; frontendDist -> ../../web/dist
```

## 9. Organizer (file management) — desktop / app-side
File ops live where the filesystem is → the Tauri desktop app (inherently desktop; browser can't).
The Tauri Rust backend reuses `core` (filename normalizer + template + move logic) and calls the
server API only for metadata/scrape resolution. Config is per-user-per-location, stored **app-side**
(Tauri local settings): `import_dir`, `export_dir`, per-type template, schedule — no backend tables.
Scheduled runs happen while the app runs; optionally the `worker` role can run the same core logic on
a box with the paths for always-on headless scheduling.

- **Flow:** scan import_dir → normalize filename → parse code → classify → ensure video (scrape via
  API if missing) → build target from the per-type template → move import → export.
- **Templates:** configurable **per video type** (jav/western/misc). Default (overridable):
  `{code}{-part?}.{ext}` → e.g. `ABC-123.mkv` / `ABC-123-1.mkv`.
- **Conflict + safety:** move + skip-if-exists by default; never silently overwrite. Override options
  exposed (overwrite / suffix). Manual-mode conflict shows a comparison (existing vs incoming: path,
  size, resolution/details) → user chooses per conflict. Manual mode does a dry-run **preview** by
  default → user approves → execute. Scheduled mode runs live and logs every move.

## 10. Testing, CI & release
- **TDD is mandatory** (see `CLAUDE.md` RULE #0): test first, never edit a test to pass code without
  explicit permission.
- **Tests:** pure `core` logic (normalizer/classifier/parser/templates) = plain unit tests, no DB.
  DB-touching code = `#[sqlx::test]` (ephemeral DB per test, auto-migrated, isolated) against a
  dedicated remote `TEST_DATABASE_URL` — that role needs `CREATEDB`. No SQLite (can't model
  `SKIP LOCKED`/arrays/jsonb); no docker.
- **CI** (`.github/workflows/ci.yml`, reusable via `workflow_call`): `backend` job = `fmt` + `clippy`
  + `cargo test` with a Postgres service container; `frontend` job (Vite+Svelte) = lint + check +
  prettier + test + build. Compile with `SQLX_OFFLINE=true` against a committed `.sqlx` cache.
- **Release** (`deploy.yml`, added in M1): one image `ghcr.io/<owner>/porna` bundling the backend
  binary + built web UI (multi-stage Dockerfile: node builds `frontend/web` → rust builds `backend`
  → final). Tags: `edge` on push to main, `:version`+`:latest` on release, multi-arch on release.
  **Tauri desktop** is a separate release job (`tauri-action`, per-OS artifacts → GitHub release).

## 11. Assumptions / open items
- Deployment: single-machine / self-hosted for now; storage is per-user-per-location so multi-user
  can be added later.
- Concrete per-site scraping parse logic: provided as each source is implemented.
- Library path for filename-parser tuning: ask the user at that step (M2).
