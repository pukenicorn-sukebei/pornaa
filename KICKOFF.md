# Kickoff

Paste `read KICKOFF.md` as your first message in a fresh session, then follow this:

1. Read **`CLAUDE.md`**, **`SPEC.md`**, and **`plan.md`** (especially the "handoff notes" at the top),
   in that order. `SPEC.md` is the source of truth; `plan.md` is the roadmap.
2. Obey **RULE #0 (strict TDD)** from `CLAUDE.md`: write the failing test **first**, then the code —
   and **never edit a test to make code pass** without explicit permission.
3. Start **M1 (foundation)** from `plan.md`. The first failing test is the `core` config resolution
   (`*_URL`-or-components), which is pure logic and needs no DB.
4. **Give me your M1 plan before writing code.**

Dev is local, **no docker** — Postgres/Redis/Byparr are remote via env (`.env`; `PG_*`/`REDIS_*` or
`*_URL`). Escalate to an Opus session for the spiky bits noted in `plan.md`.
