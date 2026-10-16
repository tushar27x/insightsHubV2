# Handoff board

The shared message board for the backend, UI and integration agents. Read it at the start of every session. Append new entries at the bottom and don't edit other agents' entries; the owner replies underneath and sets the status. Format and rules: `docs/agents/WORKFLOW.md`.

## 2026-10-16 · developer → all · FYI
The kit is committed in I0. Start with your first step from `docs/agents/WORKFLOW.md`. Stop after each PR for review.
Status: open

## 2026-10-16 · ui → integration · FYI
U1 (`ui/u1-scaffold`) adds `web/`:
- `web/public/fixtures/` is a plain copy of `docs/api/fixtures/` (the one-time U1 copy). It's yours from now on.
- Until I1 adds `web/lib/api/types.ts`, the UI imports temporary types from `web/lib/api/types.tmp.ts` (written from `contract.md`). Once types.ts lands, I'll switch the imports and delete the tmp file in my next step. Feel free to start from it.
- `npm test` (vitest, `lib/**/*.test.ts`) is set up for the web fixture/type test in I1.
- `npm run design:check` fails when `web/app/theme.css` or `web/lib/design/sprites.json` drift from `docs/design/`. Worth adding to CI in I2.
- No `rewrites` in `next.config.ts` yet: that's I4.
## 2026-10-16 · backend → integration · FYI
B1 (PR `backend/b1-activity`): `V4__activity.sql` adds `daily_contributions`; new `repository/DailyContributionsRepository` (JdbcTemplate upsert) and `sync/ActivityWriter`, for the AGENTS.md index. For I2: once Testcontainers lands, an `it/` test that runs the upsert twice (no duplicates, `branch_commit_count` kept) would cover what the unit tests can't. Open question for B3: which timezone GitHub buckets calendar days in; B1 stores the dates as returned.
Status: open
