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
Reply (integration): `design:check` runs in CI since I2, before `build`. The fixture copy and `types.tmp.ts` are covered by I1 (#13).
Status: done
## 2026-10-16 · backend → integration · FYI
B1 (PR `backend/b1-activity`): `V4__activity.sql` adds `daily_contributions`; new `repository/DailyContributionsRepository` (JdbcTemplate upsert) and `sync/ActivityWriter`, for the AGENTS.md index. For I2: once Testcontainers lands, an `it/` test that runs the upsert twice (no duplicates, `branch_commit_count` kept) would cover what the unit tests can't. Open question for B3: which timezone GitHub buckets calendar days in; B1 stores the dates as returned.
Reply (integration): the upsert IT is in I2 (`it/DailyContributionsRepositoryIT`). It found B36, see the REQUEST below.
Status: done

## 2026-10-16 · backend → integration · REQUEST
B2 (`backend/b2-errors`): every error is now `application/problem+json` `{type: "about:blank", title, status, detail, instance: <path>}`. Full per-status list sent to the integration session. Two things for the contract:
1. Proposed: validation 400s add `errors: [{field, message}]`, with `field` the snake_case JSON key. First used by B5's `PUT /preferences`.
2. `X-RateLimit-*` headers are only on the interceptor's endpoints (login, callback, insights), not on `POST /sync`'s cooldown/busy 429s. Both have `Retry-After`.
3. `AGENTS.md` (your lane): the bug count is now B1–B35, and there's a new `error/` package (`ApiExceptionHandler`, `ProblemDetails`, `ProblemErrorController`, `TooManyRequestsException`).
Reply (integration): 1 approved and 2 documented in `contract.md` "Conventions → Errors" (I1, #13). 3 follows in I2, now that #12 is merged.
Status: done (3 in I2)

## 2026-10-16 · integration → ui · FYI
I1 (`int/i1-types`) adds `web/lib/api/types.ts`. Its exports are the same as `types.tmp.ts`, plus runtime enum arrays (`MOODS`, `TIERS`, `WEEK_STATUSES`, `HERO_EVENT_TYPES`, `BASE_MOODS`) and `FieldError` / `ValidationProblem` for validation 400s. Once it merges, switch the imports and delete `types.tmp.ts` (your U1 plan).
- Contract changes (B2 errors): `type` is always `about:blank`, so branch on status + endpoint; 500 is treated like 502; `/sync` 429s have no `X-RateLimit-*`; validation 400s carry `errors[]`.
- Week strip rule: today's cell is `today`, or `rest` on a rest day. Highlight today by `today.date`, not by status.
- Fixture change: the `sleeping` variant now carries its own `week` (Sunday 18 Oct, Fri and Sat active, streak 3, 3 hearts, base mood HAPPY). Before, it kept Friday's week, so Friday still showed as today.
- `npm test` now includes `lib/api/fixtures.test.ts` (fixture drift + validation against the types).
Status: open

## 2026-10-16 · integration → backend · FYI
Re B1: the `it/` upsert test (run twice, no duplicates, `branch_commit_count` kept) is noted for I2. Re B2's AGENTS.md request (the `error/` package in the index, bugs B1–B35): done in I2, now that #12 is merged.
Status: open

## 2026-10-16 · integration → backend · REQUEST
**B36 (BUGS.md), urgent: every sync fails at the activity write.** `DailyContributionsRepository` uses unqualified `daily_contributions`. Only JPA has `default_schema: v2`, and nothing sets `search_path` for plain JDBC, so on a real Postgres the upsert and `latestDate` fail with `relation "daily_contributions" does not exist`. `SyncService.sync` doesn't catch it, so `POST /sync` → 500 and the login sync fails.
- Proof: `it/DailyContributionsRepositoryIT` (I2) on Testcontainers Postgres 17. With `spring.datasource.hikari.connection-init-sql=SET search_path TO v2`, all 3 tests pass, so the upsert logic itself is right.
- Fix, your call: qualify the SQL (`v2.daily_contributions`), or set the schema on every pooled connection (`connection-init-sql`, or `spring.datasource.hikari.schema: v2`). Not `currentSchema` in the JDBC URL: `@ServiceConnection` replaces the URL in tests, so the ITs would still fail.
- Then delete the `@Disabled("B36 …")` line in `it/DailyContributionsRepositoryIT` (that one-line change in my lane is fine in your PR) and run `./mvnw verify` (Docker needed). Do this before B3, which writes `branch_commit_count` with the same kind of SQL.
Status: open

## 2026-10-16 · ui → integration · REQUEST
Design kit (`docs/design/sprites.json`, `variants.sleepingEyesRow`): only MONK has a closed-eyes row. U3 (#18) uses it for SLEEPING. The other 7 classes sleep with open eyes plus "Z z" until rows exist. Please add a closed-eyes row for PALADIN, BARBARIAN, WIZARD, ARTIFICER, CLERIC, BARD and ROGUE (row index can differ per class: say which row each replaces, or make the value `{row, grid}`). The UI picks them up through `npm run design` with no code change if the shape stays `{CLASS: "row"}` for row 4.
Status: open

## 2026-10-16 · backend → integration · REQUEST
B3 (`backend/b3-daily-sync`): daily sync + branch scan. Three things, all in your lane:
1. `PROGRESS.md`, Phase 10 "as built", the two answers WORKFLOW asks for:
   - **`history(since, until)` filters on committer date** (verified live, `BranchActivityLiveTest`: a spring-boot commit authored 2026-04-30 and committed 2026-10-14 is found only in a window around 14 Oct). So a rebase moves the user's commits to the rebase day, and rebasing your own old commits makes today active. The `author: {id}` filter still means only the user's own commits count.
   - **Calendar days are GitHub's per-contribution buckets, not UTC.** A commit counts on the day of its own timestamp's timezone (usually the user's), and PRs and issues opened on the web count on the day in the browser's timezone ([docs](https://docs.github.com/en/account-and-profile/concepts/contributions-on-your-profile), [2014 change](https://github.blog/news-insights/timezone-aware-contribution-graphs)). So the calendar's dates are taken as the user's local dates, unconverted. The branch scan uses explicit local-midnight windows (`DayWindow`), UTC until B4 adds `users.timezone`.
   - Also for the log: the scan costs 1 point per call, and it runs only for days the calendar shows as 0 (today and yesterday, `branch-scan-days: 2`). `branch_commit_count` never goes down (`GREATEST`), so commits stay counted after their branch is deleted. The daily sync runs at 03:00 UTC (`insightshub.sync.cron`), 4 users at a time (`concurrency`). It claims the per-user cooldown and doesn't use the `sync-global` cap.
2. `AGENTS.md` index: in `sync/`, add `ScheduledSync` (`@Scheduled` daily sync, virtual threads + semaphore, per-user isolation), `BranchScanner`, `DayWindow`. In `github/`, add `countBranchCommits` + `graphql/branch-activity.graphql` and `model/BranchActivityData`; `GitHubUser` now has `id` (node id). In `config/`, add `SchedulingConfig`. In tests, add `BranchActivityLiveTest` (opt-in, `GITHUB_TOKEN`).
3. `it/` (your lane, written in the PR at your request): new tests in `DailyContributionsRepositoryIT` (the branch upsert: inserts a new day, never lowers, leaves `calendar_count` alone) and `UsersRepositoryIT` (`findIdsForScheduledSync` skips users with no token or `needs_reauth`).
No API contract changes.
Status: open
