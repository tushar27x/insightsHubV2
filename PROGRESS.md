# Insight Hub v2: Progress & Roadmap

Rewrite of the Insight Hub backend (v1: Python/FastAPI in `~/dev/insight_hub/server`, repo `tushar27x/insight_hub`) in **Spring Boot 4 + Spring AI** (repo `tushar27x/insightsHubV2`), done as a learning project.
The goal is to understand the concepts, not just to reach feature parity. Each phase lists the concepts to learn, the v1 file to look at (if any), hints, and a "done when" checklist.

**Status legend:** ⬜ not started · 🟨 in progress · 👀 PR in review · ✅ done

---

## Scope & decisions

**Why v2:** a Spring Boot project for the resume, with Spring AI replacing v1's Python LLM calls. v2 ports v1, then adds new features on top.

**What v2 adds beyond the port**
1. **Habit tracker:** a daily scheduled sync of each user's GitHub activity, kept as history.
2. **Alerts:** detected habits are sent to the user in a **daily digest email**.
3. **Archetype ranking:** archetypes become scores, and users are ranked within their archetype, globally. Archetypes are D&D-style classes tied to coding habits (decided in the PR #3 review).

**Decisions (2026-10-01)**
| Topic | Decision |
|---|---|
| Habits to detect | **Streak broken**, **no commits for N days**, **lazy commit messages** |
| Alert delivery | **Daily digest** (one email per user per day, only when there's something to report) |
| Channels | Email first, behind a `NotificationChannel` interface. Discord webhook is a stretch goal; Slack is future scope |
| GitHub data | GraphQL API for all activity data. REST only where unavoidable (OAuth code exchange, `GET /user/emails`) |
| Rankings | Global, per archetype. Friends-only ranking is future scope |
| Frontend | **Out of scope.** v2 is backend-only; test with HTTP client files / OpenAPI UI |
| RAG "Ask your Manager" (pgvector) | Stretch goal. pgvector starter stays in the pom, switched off |
| Database | Production Neon DB + remote Redis for dev (only the owner's data exists) |

**Workflow (2026-10-13)**
| Topic | Decision |
|---|---|
| Port (phases 3–9) | Implemented by Claude, **one PR per phase** against `main`, reviewed and merged by the developer |
| v2 features (2.6, 10–13) | Implemented by the developer; Claude reviews and gives hints |
| Attribution | No Claude co-author lines in commits, no Claude footer in PR descriptions |
| Bugs | Every bug is logged in [`BUGS.md`](BUGS.md); PR descriptions list the bugs found in that phase; the developer is kept informed while debugging |
| Archetype scoring | Built by Claude in PR #3 after the developer's review (code scores the class; the LLM only adds flavour in Phase 5). Phase 13 keeps the leaderboards |

---

## Current status

**Owner:** 🤖 Claude implements and opens a PR for review · 🧑 developer implements, Claude reviews

| Phase | Title | Owner | Status | PR |
|------:|-------|:-----:|:------:|:--:|
| | **Part A: Port v1** | | | |
| 0 | Setup, infra & config | 🧑 | ✅ | |
| 1 | First endpoint: health ping | 🧑 | ✅ | |
| 2 | Data model for v2 (JPA), steps 1–5 | 🧑 | ✅ | |
| 2.6 | `V3__activity.sql`: `daily_contributions`, `commits` | 🧑 | ⬜ | |
| 3 | GitHub GraphQL client | 🤖 | ✅ | [#2](https://github.com/tushar27x/insightsHubV2/pull/2) |
| 4 | Insights engine (pure Java + first tests) | 🤖 | ✅ | [#3](https://github.com/tushar27x/insightsHubV2/pull/3) |
| 5 | AI layer with Spring AI | 🤖 | ✅ | [#4](https://github.com/tushar27x/insightsHubV2/pull/4) |
| 6 | Auth: GitHub OAuth + JWT, stored token & email | 🤖 | ⬜ | |
| 7 | Sync orchestration & Redis caching | 🤖 | ⬜ | |
| 8 | Rate limiting | 🤖 | ⬜ | |
| 9 | Errors, validation, logging | 🤖 | ⬜ | |
| | **Part B: New v2 features** | | | |
| 10 | Scheduled daily sync & activity history | 🧑 | ⬜ | |
| 11 | Habit engine (3 rules) | 🧑 | ⬜ | |
| 12 | Notifications: daily digest email | 🧑 | ⬜ | |
| 13 | Archetype leaderboards (scoring done in #3) | 🧑 | ⬜ | |
| | **Part C: Ship it** | | | |
| 14 | Testing, API docs & deployment | TBD | ⬜ | |
| | **Stretch** | | | |
| S | RAG chat, Discord alerts, friends leaderboard, observability | 🧑 | ⬜ | |

**Next up:** 🤖 Phase 6 (Auth) · 🧑 Phase 2.6 (`V3__activity.sql`; `V2` is taken by the archetype classes)

---

## v1 → v2 mapping (reference)

| v1 (Python) | What it does | v2 (Spring) equivalent |
|---|---|---|
| `app/main.py` | App startup, CORS, lifespan | `InsightshubApplication`, `@Configuration` classes |
| `app/core/config.py` | env vars via dotenv | `application.yml` + `@ConfigurationProperties` |
| `app/core/security.py` | JWT + Fernet encryption | Spring Security + `JwtEncoder`/`JwtDecoder`, AES-GCM (`javax.crypto`) |
| `app/db/postgres.py` | async SQLAlchemy engine | Spring Data JPA + HikariCP (auto-configured) |
| `app/db/redis.py` | redis client | `StringRedisTemplate` / Spring Cache |
| `app/models/user.py` | `UserInsights`, `UserTemplates` | `@Entity` classes + `JpaRepository` (redesigned in Phase 2) |
| `app/services/github_service.py` | GraphQL query to GitHub | `RestClient` + `.graphql` resource file |
| `app/services/insights_service.py` | stats + archetype computation | Plain `@Service` + Java records + streams |
| `app/services/ai_service.py` | Groq + Gemini fallback, persona | Spring AI `ChatClient`, prompt templates |
| `app/api/routes.py` | `/api/auth/*`, `/api/user/insights` | `@RestController`s + a `SyncService` |
| `fastapi-limiter` | rate limit per route | `HandlerInterceptor` + Redis (or Bucket4j) |

### v1 API contract (reference)
The frontend is out of scope, but keeping v1's response shape means the v1 Next.js client could be pointed at v2 later.
- `GET /api/auth/login`: redirects to GitHub (sets `oauth_state` cookie)
- `GET /api/auth/callback?code&state`: syncs data, redirects to `FRONTEND_URL/dashboard?token=<jwt>`
- `GET /api/auth/logout`: redirects to `FRONTEND_URL`
- `GET /api/user/insights`: `Authorization: Bearer <jwt>`, returns **snake_case** JSON:
  `user_name, archetype, stats{...}, display_json, weekly_review, craft_review, updated_at`

---

# Part A: Port v1

## Phase 0: Setup, infra & config ✅

**Concepts:** Spring Boot auto-configuration, starters, `application.yml`, profiles, externalized config.

**Hints**
1. ⚠️ **Version check first.** Your `pom.xml` uses Spring Boot **4.1.1** with Spring AI **1.0.0**. Spring AI 1.0.x was built against Boot 3.x (Spring Framework 6, Jackson 2). Boot 4 moved to Framework 7 and Jackson 3. Find out which Spring AI line supports Boot 4 and align the versions. Otherwise you'll hit confusing auto-config errors later.
2. The starters you added will try to connect to Postgres, Redis and OpenAI on startup. Make both reachable (local Docker or remote services).
3. Rename `application.properties` → `application.yml`. Read secrets from env vars with `${GITHUB_CLIENT_ID}` placeholders. Never commit secrets. Add a git-ignored `.env` or `application-local.yml`.
4. Create a typed config class: a Java `record` annotated with `@ConfigurationProperties(prefix = "insightshub")` holding `frontendUrl`, `backendUrl`, `jwtSecret`, `encryptionKey`, etc. Figure out how to enable it (`@ConfigurationPropertiesScan`).
5. Run `git init`. You'll want history for this.

**Done when**
- [x] Spring Boot / Spring AI versions are compatible
- [x] Postgres (Neon) + Redis (remote) reachable from the app
- [x] `./mvnw spring-boot:run` starts cleanly
- [x] Config values come from env vars through a typed `@ConfigurationProperties` record

---

## Phase 1: First endpoint: health ping ✅

**v1 reference:** `routes.py` → `GET /api/` (sets and gets a Redis key, runs `SELECT 1`)

**Concepts:** `@RestController`, `@GetMapping`, constructor injection, beans, `JdbcTemplate`, `StringRedisTemplate`, Actuator.

**Hints**
1. Unlike FastAPI's `Depends(get_db)`, Spring injects dependencies through the constructor. Use `final` fields + one constructor (or Lombok `@RequiredArgsConstructor`).
2. Return a `Map` or a `record` from the controller. Jackson serializes it for you.
3. Set a global `/api` prefix instead of repeating it on every controller (`server.servlet.context-path`, or a path prefix config). Think about which one also affects Actuator URLs.
4. Afterwards, look at `/actuator/health`. It already checks DB and Redis. That's the "Spring way" to do what v1's ping did.

**Done when**
- [x] `GET /ping` returned `{"redis":"pong","jdbc":1}`, then replaced by `/actuator/health` with details
- [x] You can explain what a bean is and who creates your controller

---

## Phase 2: Data model for v2 (JPA) 🟨

**v1 reference:** `models/user.py`, `db/postgres.py`. v2 needs more than v1's two tables, because habits need **history** (v1 overwrote `stats_json` on every sync) and rankings need **scores**.

**Concepts:** `@Entity`, `@Id`, `@OneToOne` + `@MapsId`, composite keys (`@EmbeddedId` / `@IdClass`), `@ManyToOne`, unique constraints, JSONB columns, `ddl-auto` vs Flyway migrations, `@Transactional`.

**Tables to design** (names are suggestions):
| Table | Purpose | Notes |
|---|---|---|
| `users` | GitHub id (PK), login, **email**, **encrypted GitHub token**, preferences (`alerts_enabled`, `no_commit_days`, `public_profile`), `last_synced_at` | Preferences could be a separate table. Your call |
| `insights` | Latest computed stats JSON, AI texts (year in review, weekly, craft), archetype | Replaces v1's `user_templates` |
| `daily_contributions` | `(user_id, date)` → `count` | Composite key. One row per day, backfilled from the contribution calendar |
| `commits` | `sha` (unique), user, repo, message, `committed_at` | Feeds the lazy-message rule. `sha` makes inserts idempotent |
| `habit_alerts` | user, type, detected date, details JSON, `digest_id` (null = not sent yet) | Unique `(user_id, type, detected_on)` so the same alert can't be created twice |
| `digests` | user, sent-at, channel, status | Log of what was actually sent |
| `archetype_scores` | user, archetype, score, `algo_version`, computed-at | Used in Phase 13 |

**Hints**
1. ⚠️ **v1 is still live** (insight-hub-nu.vercel.app) on the same Neon DB. Don't let `ddl-auto: update` touch v1's `user_insights`/`user_templates`. Use new table names, or better, a separate Postgres **schema** for v2 (look at `spring.jpa.properties.hibernate.default_schema`).
2. GitHub IDs aren't generated by you, so no `@GeneratedValue` on `users`. Use `Long`.
3. For a table whose primary key is also a foreign key to `users`, like `insights`, look up `@MapsId`.
4. JSON columns: `@JdbcTypeCode(SqlTypes.JSON)` on a `Map<String,Object>` or on a record → `jsonb`.
5. With 7 tables, composite keys and unique constraints, this is the point where **Flyway** pays off (versioned SQL in `db/migration/V1__init.sql`, then `ddl-auto: validate`). Strongly recommended here, since it's what real teams do.
6. Timestamps: `Instant` for moments; `LocalDate` for "a day" in `daily_contributions`. Pick one timezone policy (UTC) and write it down.
7. Prefer `@ManyToOne(fetch = LAZY)` from child to `users`, and avoid `@OneToMany` collections on `User` unless you need them. Find out what the N+1 problem is.

**Done when**
- [ ] All tables exist in v2's own schema (via Flyway or entities), and v1's tables are untouched
- [x] Repositories exist and `findById` works for `users`
- [ ] Inserting the same `(user_id, date)` or commit `sha` twice is impossible at the DB level

---

## Phase 3: GitHub GraphQL client ✅

**v1 reference:** `services/github_service.py`

**Concepts:** `RestClient` (the synchronous HTTP client that replaced `RestTemplate`), resources on the classpath, Jackson deserialization into records, error handling, GraphQL variables and rate limits.

**Hints**
1. Put the query in `src/main/resources/graphql/user-insights.graphql` and load it with `@Value("classpath:...") Resource`.
2. Build one `RestClient` bean with `baseUrl("https://api.github.com")` using `RestClient.Builder` (auto-configured, so inject it).
3. Big design choice: map the response into **nested Java records** (type-safe, more code) or into a `JsonNode` (quick, like Python dicts). Recommendation: use records. That's where Java pays off. Use `@JsonIgnoreProperties(ignoreUnknown = true)`.
4. GraphQL returns HTTP 200 even on errors. Check the `errors` field yourself and throw a custom exception.
5. **Extend v1's query for v2:**
   - commit history nodes: add `oid` (the SHA) and `committedDate`, not just `message`
   - add a top-level `rateLimit { cost remaining resetAt }` and log it on every call
   - `contributionsCollection(from:, to:)` accepts a date range. The scheduled sync (Phase 10) can fetch only the recent days instead of the whole year
6. Rate limits: GraphQL gives **5,000 points per hour per user token**. Each user syncs with their own token, so the budget isn't shared between users.
7. Test it manually with a personal access token before wiring OAuth (e.g. a temporary dev endpoint or a `CommandLineRunner`).

**Done when**
- [x] Your GitHub profile is fetched and mapped into a record tree, including commit SHAs and dates
- [x] GraphQL errors raise a meaningful exception
- [x] `rateLimit.cost` / `remaining` show up in the logs

---

## Phase 4: Insights engine (pure Java + first tests) ✅

**v1 reference:** `services/insights_service.py` (`calculate_user_insights`, `determine_archetype`)

**Concepts:** Streams (`map`, `flatMap`, `sorted`, `Collectors.groupingBy`/`counting`), `LocalDate`/`DayOfWeek`, records, enums, **JUnit 5 + AssertJ** unit tests.

**Hints**
1. This is pure logic with no Spring needed. Keep it that way: input is the GitHub record from Phase 3, output is an `InsightsResult(Stats stats, Archetype archetype)` record.
2. Model the archetype as an `enum` with a display label (`"🕵️ The Bug Hunter"`).
3. Port v1's first-match archetype rules as they are for now. Phase 13 replaces them with scores, so keep the archetype logic in its own class, behind an interface, so it's easy to swap.
4. The "flatten weeks → days, sort by date, take last N" logic appears 3 times in v1. Write it once. You'll reuse it for `daily_contributions` too.
5. Language counting: `flatMap` over repos → languages, then `groupingBy(identity(), counting())`, then sort by value.
6. Write unit tests **here first**. There's no DB and no HTTP, so it's a perfect TDD target. Cover each archetype branch.
7. Note: the generated `InsightshubApplicationTests.contextLoads()` starts the full context, so it needs DB/Redis/Groq. Decide how to handle it (Phase 14: Testcontainers).

**Done when**
- [x] Output matches v1's `stats` JSON shape for the same input
- [x] Unit tests cover all 5 archetypes + heatmap edge cases (< 7 days)
- [x] **Review change:** archetypes redesigned as 8 scored D&D classes (see below)

**Archetype redesign (PR #3 review)**
- v1's five archetypes were first-match thresholds on raw totals, and some weren't habits at all. They're replaced by **8 classes**, each tied to one habit: 🛡️ Paladin (streaks), 🪓 Barbarian (huge PRs, bursts), 🧙 Wizard (languages), ⚙️ Artificer (infra files), ✨ Cleric (issues, fix commits), 🎻 Bard (PRs, audience, docs), 🧘 Monk (small PRs, descriptive messages), 🗡️ Rogue (weekends).
- `HabitProfile.from(user, days)` derives 15 signals (shares 0..1); commit signals use **only the user's own commits**. `ScoringArchetypeClassifier` gives every class a 0–100 score (weighted signals scaled between a floor and a cap); highest wins, ties go to declaration order, all-zero → Rogue. `ALGO_VERSION = 1`.
- Why code decides and not the LLM: stable classes for leaderboards, no prompt injection through READMEs or commit messages, works when the AI is down, unit-testable, no extra LLM call. The LLM writes the "character sheet" (title, lore) in Phase 5.
- `V2__archetype_classes.sql` swaps the CHECK constraint and maps old values (dry-run on Neon in a rolled-back transaction). Scores aren't persisted yet: that's Phase 7 / Phase 13's `archetype_scores` table.
- `InsightsEngineLiveTest` (opt-in) prints a real profile's signals and scores for calibration.

---

## Phase 5: AI layer with Spring AI ⭐ ✅

**v1 reference:** `services/ai_service.py`

**Concepts:** `ChatClient` fluent API, system vs user messages, `PromptTemplate` (StringTemplate `.st` files), OpenAI-compatible providers, multiple model beans + `@Qualifier`, structured output (`.entity(...)`), fallback strategies, parallelism with `CompletableFuture` / virtual threads.

**Hints**
1. **You don't need a Groq SDK.** Groq exposes an OpenAI-compatible API (already configured in Phase 0). Gemini also has an OpenAI-compatible endpoint, which is useful for the fallback.
2. Put `MANAGER_PERSONA` in `resources/prompts/persona.st` and each review prompt in its own `.st` file with `{variables}`. Load them as `Resource`s.
3. Build a `ChatClient` bean with `.defaultSystem(persona)` so every call carries the persona.
4. Fallback: you'll need **two** chat clients (primary + fallback). Auto-config gives you one, so read up on creating a second `OpenAiChatModel` manually and telling them apart with `@Qualifier`. Wrap the call in try/catch → fallback → static message, the same as v1. (Stretch: Spring Retry or Resilience4j.)
5. `generate_repo_summaries` returns a `Map<String,String>`. Try `.call().entity(new ParameterizedTypeReference<...>())` to get structured output in **one** call instead of 3.
6. v1 used `asyncio.gather` for 4 AI calls in parallel. In Java: `CompletableFuture.supplyAsync(..., executor)` + `allOf`. Enable virtual threads (`spring.threads.virtual.enabled=true`) and use a virtual-thread executor. Handle each future's exception separately (like `return_exceptions=True`).
7. Per-call options (`max_tokens`, model override for the cheaper 8B model) go through `.options(...)`.
8. Design the AI service so Phase 12 can reuse it. The digest email's intro will be written in the same mentor persona.
9. **Character sheet** (from the PR #3 review): a 5th generator that takes the scored class, `ArchetypeScores` and `HabitProfile` and writes a D&D-style title and short lore. The class is a **fixed input**; the LLM can't change it. Use structured output (`.entity(CharacterSheet.class)`) and a static fallback built from `Archetype.getLabel()` / `getHabit()`.

**Done when**
- [x] All 4 generators work against Groq (+ the character sheet), checked live by `AiLiveTest`
- [x] Killing the Groq key falls back to the next provider. Proven live with a dead key → a second Groq provider; Gemini is wired but switched off until `GEMINI_API_KEY` is set
- [x] The calls run in parallel: 5 generators in ~2 s live; unit test: 5 × 400 ms tasks finish in < 1.2 s

**As built**
- `ai` package: `LlmChain` (providers in order → first usable answer, never throws), `LlmProvider` (client + model per `ModelTier` + reasoning effort), `AiConfig` (Groq from auto-config + optional hand-built fallback `OpenAiChatModel`), `AiReviewer` (5 generators with static fallbacks), `AiInsightsService` (parallel on Boot's virtual-thread `applicationTaskExecutor`, per-task timeout and crash isolation), `AiProperties` (`insightshub.ai.*`).
- A provider chain instead of `@Qualifier` beans: the fallback is optional, and the try-next logic lives in one place instead of every generator.
- Repo summaries: one structured call (`.entity(RepoSummaries.class)`) instead of v1's 3. Invented repo names are dropped, skipped repos get v1's default text, and READMEs are marked as untrusted data in the prompt.
- Character sheet: a title that doesn't name the scored class is rejected (next provider, then the static sheet).
- **Models changed:** v1's Llama models are retired on Groq (B26). Now `openai/gpt-oss-120b` / `gpt-oss-20b` with `reasoning_effort: low` and token headroom (B27).
- **Persona changed (developer's call):** v1's roast and "tough love" manager are gone. v2 is a habit tracker, so every generator speaks as a supportive senior engineer: specific strengths, one habit to grow, one small next step, no sarcasm or invented praise. The roast became the **year in review**. `AiReviewerTest` fails if roast-style wording comes back into a prompt.
- Not done here (Phase 7): persisting the AI texts (the character sheet needs columns, and `insights.roast` gets renamed to `year_in_review`), and attaching summaries to `ranked_repos`.


---

## Phase 6: Auth: GitHub OAuth + JWT, stored token & email ⬜

**v1 reference:** `routes.py` (`/auth/login`, `/auth/callback`, `/auth/logout`), `core/security.py`

**Concepts:** Spring Security filter chain, `SecurityFilterChain` bean, stateless sessions, OAuth2 authorization code flow, OAuth scopes, JWT (Nimbus: `JwtEncoder`/`JwtDecoder`), `@AuthenticationPrincipal`, symmetric encryption (AES-GCM).

**Hints**
1. You need new dependencies: `spring-boot-starter-security`, and `spring-boot-starter-oauth2-resource-server` for JWT validation (it brings Nimbus).
2. **Two approaches for the GitHub login part:**
   - **(a) Hand-rolled, like v1:** controller builds the authorize URL, stores `state` in a cookie, and the callback exchanges the code with `RestClient`. Easiest to reason about.
   - **(b) `spring-boot-starter-oauth2-client`:** Spring handles state/redirect/code exchange, and you plug in an `AuthenticationSuccessHandler` that issues your JWT.
   - Recommendation: start with **(a)** to understand the flow, then refactor to **(b)** as a learning exercise.
3. **Scopes change in v2.** Add `user:email` and fetch the primary, verified address with REST `GET /user/emails`. GraphQL only exposes the *public* email. Also think about `repo`: v1 asked for it, but it grants full read/write to private repos, and v2 now **stores** the token long-term. Public data only (no `repo`), or private activity too? Decide and write down the trade-off.
4. **Store the GitHub token encrypted.** The scheduled sync (Phase 10) runs without the user present, so it needs it. Fernet is Python-specific; use **AES-GCM** via `javax.crypto` (random 12-byte IV prepended, Base64 encoded, 32-byte key from `ENCRYPTION_KEY`). Never log it, and never return it from an API.
5. JWT issuing: define `JwtEncoder`/`JwtDecoder` beans using an HMAC secret (`NimbusJwtEncoder` + `ImmutableSecret`). Claims: `sub=githubId`, `name=login`, `exp=24h`.
6. **Backend-only:** there's no frontend to redirect to. Have the callback return the JWT as JSON (or a tiny HTML page showing it) for now, and keep the `FRONTEND_URL` redirect as something you can switch on later.
7. Protect `/user/**`, permit `/auth/**`, health and the unsubscribe link (Phase 12). Use `SessionCreationPolicy.STATELESS` and disable CSRF (Bearer tokens only).
8. Read the user in controllers with `@AuthenticationPrincipal Jwt jwt` → `jwt.getSubject()`.

**Done when**
- [ ] The login round-trip yields a JWT
- [ ] `users` has your email and an encrypted token that decrypts back to a working GitHub token
- [ ] `GET /user/insights` returns 401 without a token and 200 with one

---

## Phase 7: Sync orchestration & Redis caching ⬜

**v1 reference:** the big body of `callback()` + `get_user_insights()` in `routes.py`

**Concepts:** Service layer design, `@Transactional` boundaries, Spring Cache abstraction (`@EnableCaching`, `@Cacheable`, `@CacheEvict`), `RedisCacheManager` config, TTLs, JSON serialization in Redis.

**Hints**
1. The v1 callback does far too much. Extract a `SyncService.sync(userId)`: fetch → compute → AI → upsert → evict cache. **Both** the login callback and the Phase 10 scheduler will call it, so it must not depend on anything from the HTTP request.
2. **Don't hold a DB transaction open during GitHub + AI calls** (they take seconds). Do the slow work first, then persist inside a short `@Transactional` method. Beware: `@Transactional` on a method called from *within the same class* doesn't work. Find out why (proxies!).
3. Build the insights response DTO as a record in an `InsightsQueryService.getInsights(userId)` annotated `@Cacheable(value="user_insights", key="#userId")`. Configure a 24h TTL in a `RedisCacheConfiguration` bean with a JSON serializer.
4. The migration for this phase adds the character-sheet columns and renames `insights.roast` to `year_in_review` (Phase 5 replaced the roast). Rename the `Insights` entity field to match.
5. `@CacheEvict(value="user_insights", key="#userId")` on the sync's persist step replaces v1's manual `redis.delete`.
6. v1 swallows sync errors so login still works. Keep that behavior, but log properly.
7. **Phase 5 follow-ups** (these calls start running for real users here):
   - The 90 s AI task timeout stops waiting but doesn't cancel the HTTP call (worst case about 2 × 30 s per provider). Cancel it, or make the timeouts add up to less than the task timeout.
   - Log token usage per call (`.call().chatResponse()` metadata instead of `.content()`), since Phase 10 syncs everyone daily.
   - Mark commit messages as untrusted data in `craft-review.st`, like READMEs already are.
   - Gemini fallback is untested against the real endpoint: needs `GEMINI_API_KEY` (and maybe `GEMINI_MODEL`).

**Done when**
- [ ] Second `GET /user/insights` is served from Redis (check with `redis-cli KEYS *`)
- [ ] A fresh sync evicts the cache

---

## Phase 8: Rate limiting ⬜

**v1 reference:** `RateLimiter(times=5, seconds=60)` on auth routes, 20/60s on insights

**Concepts:** `HandlerInterceptor` / `OncePerRequestFilter`, `WebMvcConfigurer`, custom annotations + reflection, Redis `INCR` + `EXPIRE`.

**Hints**
1. Create a `@RateLimit(requests = 5, seconds = 60)` annotation for controller methods.
2. In a `HandlerInterceptor`, check `handler instanceof HandlerMethod`, read the annotation, and build a key `rl:{ip-or-userId}:{route}`. Use Redis `INCR`, and set `EXPIRE` on the first hit. If the count is over the limit, return 429.
3. Alternative: the Bucket4j library (token bucket). Build your own first, then compare.

**Done when**
- [ ] The 6th login within 60s returns 429 with a `Retry-After` header

---

## Phase 9: Errors, validation, logging ⬜

**Concepts:** `@RestControllerAdvice`, `ProblemDetail` (RFC 9457), custom exceptions, Bean Validation (`@Valid`, `@NotBlank`), SLF4J (`@Slf4j`), log levels per package.

**Hints**
1. Replace every `print(...)` from v1 with proper log levels. **Never log tokens** (v1 printed the GitHub token response!).
2. Map exceptions to responses in one place: `GitHubApiException` → 502, `UserNotFoundException` → 404, and so on.
3. Validate request bodies, such as the preferences update in Phase 12, with `@Valid` and constraint annotations.
4. CORS only matters once a browser frontend calls the API. Skip it until then, but know it goes through Spring Security's `.cors(...)`.

**Done when**
- [ ] All errors return consistent `ProblemDetail` JSON
- [ ] No secrets in logs

---

# Part B: New v2 features

## Phase 10: Scheduled daily sync & activity history ⬜

**Goal:** every opted-in user gets synced once a day without logging in, and their activity is stored as history.

**Concepts:** `@EnableScheduling`, `@Scheduled(cron = ...)` with the cron expression in config, upserts (`INSERT ... ON CONFLICT`), virtual threads + bounded concurrency, error isolation per user, `java.time.Clock` injection.

**Hints**
1. Put the cron expression in `application.yml` (`insightshub.sync.cron`) and reference it with `${...}`, so you can run it every minute while developing.
2. The job: load users with a stored token → for each, call `SyncService.sync(userId)` → write `daily_contributions` and `commits`. **One user failing must not stop the others** (try/catch per user, then log and continue).
3. First sync per user: backfill a year of `daily_contributions` from the contribution calendar. Later syncs: only the recent window (`from:`/`to:`).
4. Upserts: Spring Data's `save()` does a SELECT and then an INSERT/UPDATE. For bulk daily rows, compare that with a native `INSERT ... ON CONFLICT DO UPDATE` query. Which one is better here, and why?
5. Run users in parallel on virtual threads, but limit concurrency (a `Semaphore`, or a fixed-size executor). Being polite to GitHub's secondary rate limits matters more than speed.
6. A 401 from GitHub means the token was revoked. Mark the user `needs_reauth` and stop syncing them, rather than failing every day.
7. Inject a `Clock` bean instead of calling `LocalDate.now()`. Your tests in Phase 11 will thank you.
8. Ask yourself: what happens if two instances of the app run the job at once? (Answer later with ShedLock, stretch goal.)
9. When the job finishes for a user, publish an event (`ApplicationEventPublisher`, e.g. `UserSyncedEvent`). Phase 11 listens for it.

**Done when**
- [ ] The job runs on schedule and fills `daily_contributions` + `commits` for your user
- [ ] Re-running it creates no duplicate rows
- [ ] A broken token for one user doesn't stop the others

---

## Phase 11: Habit engine (3 rules) ⬜

**Goal:** after each sync, detect the three habits and store them as pending alerts.

**Concepts:** Strategy pattern via Spring (`List<HabitRule>` injection), pure functions + unit tests, `@EventListener` / `@TransactionalEventListener(phase = AFTER_COMMIT)`, idempotency through unique constraints.

**The rules**
| Rule | Fires when | Avoid nagging |
|---|---|---|
| **Streak broken** | Yesterday = 0 contributions, and the streak before it was ≥ a minimum length (e.g. 3 days) | Fires once per broken streak |
| **No commits for N days** | The last N days are all 0 (N = user preference, default 3) | Fire on day N, then at most once a week while it continues |
| **Lazy commit messages** | Among the last day's (or week's) commits, the share of "lazy" messages is above a threshold | Needs a minimum number of commits, so one bad commit doesn't trigger it |

**Hints**
1. Define `interface HabitRule { Optional<HabitAlert> evaluate(UserActivity activity); }` and make each rule a `@Component`. Then inject `List<HabitRule>` and Spring hands you **all** of them. Adding a fourth rule later means adding one class.
2. Keep the rules pure: build a `UserActivity` record (recent daily counts, recent commits, preferences, today's date) from the DB, then pass it in. No repositories inside the rules → easy unit tests.
3. "Lazy" message heuristic, as a starting point: very short, or matching words like `wip`, `fix`, `update`, `changes`, `asdf`, `.`. Write it as a unit-tested function. **Spring AI option:** classify the messages with `.entity(...)` structured output and have the model say *why* they're lazy. Heuristic first, AI as an upgrade.
4. Listen for `UserSyncedEvent`. Find out the difference between `@EventListener` and `@TransactionalEventListener(AFTER_COMMIT)`, and why the second matters here.
5. Save detected alerts to `habit_alerts`. The unique `(user_id, type, detected_on)` constraint makes re-runs harmless. Catch the duplicate-key error or check first, and know which is better.
6. Unit test each rule with fixed dates (e.g. a streak broken on a Monday, N-day gaps across a month boundary).

**Done when**
- [ ] Each rule has unit tests for "fires" and "doesn't fire"
- [ ] Syncing creates `habit_alerts` rows, and re-syncing doesn't duplicate them

---

## Phase 12: Notifications: daily digest email ⬜

**Goal:** once a day, each user with pending alerts gets **one** email that summarises them in the mentor persona.

**Concepts:** `JavaMailSender` + `MimeMessageHelper`, HTML email templates (Thymeleaf), Strategy pattern (`NotificationChannel`), delivery guarantees (at-least-once vs at-most-once), signed unsubscribe links, a preferences API.

**Hints**
1. `interface NotificationChannel { void send(User user, Digest digest); }` with an `EmailChannel` implementation. Discord comes later as a second implementation (stretch).
2. Digest job (`@Scheduled`, after the sync job): for each user with pending `habit_alerts` (`digest_id IS NULL`) → build one digest → send → mark the alerts as sent. **No alerts, no email.**
3. Think through failure: if sending succeeds but marking fails, the user gets a duplicate tomorrow. If you mark first and sending fails, the alert is lost. Pick one and write down why (a digest that occasionally repeats is usually the lesser evil).
4. Content: Spring AI writes a short intro in the mentor persona from the alert list (reuse Phase 5). **The email must still go out if the AI call fails**, so have a plain template fallback.
5. HTML: add `spring-boot-starter-thymeleaf` and render `templates/email/digest.html` with `TemplateEngine` into a string. That's server-side rendering for email, not a web frontend.
6. SMTP for development: a sandbox inbox (e.g. Mailtrap) or a Gmail app password. Use a real provider (Resend, Brevo, SES) later. All of them are configured with `spring.mail.*` properties.
7. Unsubscribe: every email includes a link like `/api/notifications/unsubscribe?token=...`. The token should be signed (you already have JWT tooling) and the endpoint public.
8. Preferences API: `GET`/`PUT /api/user/preferences` (alerts on/off, N for "no commits", leaderboard visibility). Validate the input (Phase 9).

**Done when**
- [ ] One digest arrives with all pending alerts, and none is sent when there's nothing to report
- [ ] The unsubscribe link turns alerts off
- [ ] An AI outage still produces an email

---

## Phase 13: Archetype scoring & leaderboards ⬜

**Goal:** archetypes become **scores**, and users are ranked within their archetype, globally.

> ✅ **Scoring is done** (PR #3 review): `HabitProfile` + `ScoringArchetypeClassifier` give 8 D&D classes 0–100 scores with `ALGO_VERSION`. Hint 1 is covered and hint 4 uses fixed caps. What's left here: storing scores (hints 2–3), leaderboards (5–7) and the seeder (8).

**Concepts:** normalisation and scoring design, versioned algorithms, SQL window functions (`RANK() OVER (PARTITION BY ...)`, supported in Hibernate 6+ HQL), Spring Data `Pageable`, Redis **sorted sets** (`ZADD`, `ZREVRANK`, `ZREVRANGE`), profiles for seed data.

**Hints**
1. ~~Replace the first-match rules (Phase 4) with a scorer~~ (done in #3): compute a 0–100 score for **each** archetype from normalised metrics (e.g. issues ratio → Bug Hunter, PR ratio → Open Source Hero, infra files → Architect, weekend share → Weekend Warrior). Primary archetype = highest score, with a fixed tie-break order.
2. Store the breakdown (which metric contributed what) as JSON, so you can explain a score, and maybe let the AI explain it.
3. Store `algo_version` with every score. When the formula changes, recompute everyone so old and new scores never mix in one ranking.
4. Normalisation: with few users, use fixed caps (e.g. 1,000 commits → 100). With many, percentiles. Start with caps.
5. Ranking in Postgres: `RANK() OVER (PARTITION BY archetype ORDER BY score DESC)`. Write it in HQL or as a native query. Endpoints: `GET /api/leaderboard?archetype=...&page=...` (paged), and `GET /api/user/rank` (your rank + percentile).
6. Mirror into Redis sorted sets (`leaderboard:{archetype}`): `ZADD` on recompute, `ZREVRANK` for "my rank", `ZREVRANGE ... WITHSCORES` for the top N. Postgres stays the source of truth, so rebuild Redis from it if it's empty.
7. Only users with `public_profile = true` appear on the leaderboard.
8. **Cold start:** you're the only user. Add a `dev`-profile seeder (`@Profile("dev")` + `CommandLineRunner`) that inserts fake users flagged `is_demo`, so rankings show something, and make sure it never runs in prod.

**Done when**
- [ ] Every user has scores for all archetypes, and the primary archetype is derived from them
- [ ] Leaderboard + "my rank" endpoints work from both Postgres and Redis and agree with each other
- [ ] Changing `algo_version` triggers a full recompute

---

# Part C: Ship it

## Phase 14: Testing, API docs & deployment ⬜

**Concepts:** `@SpringBootTest` + **Testcontainers** + `@ServiceConnection`, `@WebMvcTest` slice tests, `MockMvc`, security tests, OpenAPI docs, Docker images (`spring-boot:build-image`), production profile.

**Hints**
1. Add Testcontainers (Postgres + Redis) and fix `contextLoads()`. Mock the AI and GitHub clients in integration tests.
2. `@WebMvcTest` your controllers with mocked services, and test the security rules (401/200).
3. Backend-only means the API docs are your UI. Add `springdoc-openapi` (check which version supports Boot 4) for Swagger UI, and keep `.http` request files in the repo.
4. snake_case JSON for v1 compatibility: a global `SNAKE_CASE` naming strategy, or `@JsonProperty` per field. Know the trade-off.
5. Build an image with `./mvnw spring-boot:build-image`, add `application-prod.yml`, and deploy (v1 used Koyeb).

**Done when**
- [ ] `./mvnw verify` is green with integration tests
- [ ] Swagger UI documents every endpoint
- [ ] The app runs in production, and the daily sync + digest fire there

---

## Stretch goals ⬜

- **S1. "Ask your Manager" (RAG chat):** embed READMEs, commit messages and repo summaries into pgvector during sync (`userId` metadata). `POST /user/ask` → `ChatClient` + `QuestionAnswerAdvisor` filtered by user. Needs an embedding provider (Groq has none), and re-enabling `spring.ai.model.embedding` and `spring.ai.vectorstore.type`.
- **S2. Discord alerts:** `DiscordWebhookChannel` as a second `NotificationChannel`. The user saves a webhook URL, and you POST JSON to it. Slack later.
- **S3. Friends leaderboard:** rank among the people you follow on GitHub (`following` in GraphQL).
- **S4. Observability:** Actuator + Micrometer metrics for AI calls, fallback rate, sync duration, digests sent.
- **S5. Multi-instance safety:** ShedLock so scheduled jobs run once across instances.

---

## Learning log

One entry per session: what was built, what was learned, open questions. Bug details live in [`BUGS.md`](BUGS.md) (B-numbers).

### 2026-09-27: Kickoff
- Scanned the v1 backend (`~/dev/insight_hub/server`) and the v2 skeleton; created this roadmap and `.claude/CLAUDE.md`.

### 2026-09-27: Phase 0 complete ✅
- Spring AI 2.0.0 on Boot 4 (B1). `.env` loaded with `spring.config.import` (B4). `@Validated` record `InsightsHubProperties` (B2, B3) fails fast on blank values. `.gitignore` covers secrets (B5).
- Groq as an OpenAI-compatible chat provider at `https://api.groq.com/openai/v1` (B8). Unused OpenAI models, embeddings and pgvector off via `spring.ai.model.*: none` / `spring.ai.vectorstore.type: none` (B6, B7).
- Decision: production Neon DB + remote Redis for dev (only the owner's data exists).
- Learned: starters switch features on, properties switch them off; unknown YAML keys are silently ignored; the real cause is at the first `WARN`/`Caused by`.

### 2026-10-01: Scope change: v2 features defined
- v2 = port + habit tracker + daily digest alerts + archetype ranking; backend-only; RAG is a stretch goal. See "Scope & decisions".
- Consequences for the port: Phase 3 fetches commit SHAs/dates and logs `rateLimit`; Phase 6 adds `user:email` and stores the GitHub token encrypted; Phase 7's `SyncService` must work without an HTTP request. v1 is still live on the same Neon DB, so v2 uses its own schema.

### 2026-10-01: Phase 1 complete ✅
- Built `/ping` (constructor injection, `StringRedisTemplate` + `JdbcTemplate`), then replaced it with `/actuator/health` + `show-details: always` (switch to `when-authorized` in Phase 6). Redis host fixed (B9).
- Learned: Actuator aggregates `HealthIndicator` beans from each starter; Redis connects lazily, Postgres eagerly.
- Carried to Phase 6: the `/api` prefix decision.

### 2026-10-01 → 10-06: Phase 2, steps 1–5
- Schema review: `users` (not `user`, a reserved word); `insights.user_id` as PK + FK (`@MapsId`); no stored `global_rank` (derive it in Phase 13); archetype as `varchar` + CHECK on `insights`; daily digest only.
- Flyway (`spring-boot-starter-flyway` + `flyway-database-postgresql`) owns the `v2` schema; `ddl-auto: validate`. `V1__users_and_insights.sql` applied to Neon; v1's `public` schema untouched.
- Archetype names are fixed by the CHECK constraint: `BUG_HUNTER, OPEN_SOURCE_HERO, SOFTWARE_ARCHITECT, WEEKEND_WARRIOR, CODE_CRUSADER` (replaced by 8 D&D classes in V2, PR #3).
- Entities `Users`, `Insights`, enum `Archetype`; `UsersRepository` (`findByAlertsEnabledTrue`, `@Query findDueForSync`), `InsightsRepository` (`findById` with `@EntityGraph`). Bugs along the way: B10–B18.
- Idempotent `dev`-profile round-trip runner (`DevDataRunner`) proved: `@MapsId`, `jsonb` round trip, enum vs CHECK, generated `createdAt`, fetch graph. `open-in-view: false`.
- Correction: `columnDefinition = "text"` is *not* required for `String` → `text`; Hibernate 7 accepts it (only documentation).
- Rules learned: DB defaults don't apply to explicit NULLs; `save()` persists or merges based on the id, so load existing rows before updating; never return entities from controllers (DTOs only).

### 2026-10-13: Working agreement changed
- **Port (phases 3–9):** implemented by Claude, one PR per phase for the developer to review.
- **Phase 2 step 6 and v2 features (phases 10–13):** implemented by the developer, with Claude reviewing and hinting.
- No Claude attribution in commits or PR descriptions. Bugs are logged in `BUGS.md`, and each PR lists the bugs found while building it.

### 2026-10-13: Phase 3 built (🤖)
- `github` package: `GitHubGraphQlClient` (`fetchUserInsights(token, login[, from, to])`), `GitHubClientConfig` (`RestClient` with base URL + `User-Agent`), typed records in `github.model`, `GitHubApiException` / `GitHubAuthException` (401, for Phase 10's `needs_reauth`). `Clock` bean in `config/ClockConfig`.
- Query `graphql/user-insights.graphql` = v1's query + `rateLimit`, commit `oid`/`committedDate`/`author.user.login`, and a `from`/`to` window (default: last 365 days; GitHub's max is 1 year).
- Tests: `GitHubGraphQlClientTest` (9 unit tests, `MockRestServiceServer`, fixed `Clock`); `GitHubGraphQlClientLiveTest` (opt-in, needs `GITHUB_TOKEN`). Live: the query costs **1 point**.
- Bug log: B19.
- ⚠️ For 2.6/11: `history` returns the default branch's last 5 commits by **anyone**. Filter by `author.user.login` (null when the commit email isn't linked to an account) to get the user's own commits.

### 2026-10-13: Phase 4 built (🤖)
- `insights` package: `InsightsEngine.calculate(GitHubUser)` → `InsightsResult(UserStats, Archetype)`. `UserStats` serializes with v1's snake_case keys (`@JsonNaming`). The archetype rules (`RuleBasedArchetypeClassifier`) sit behind `ArchetypeClassifier`, so Phase 13 can replace them.
- **Golden test against v1:** ran v1's actual Python `calculate_user_insights` on a fixture → `insights/v1-expected-insights.json`; `InsightsEngineParityTest` asserts the Java stats JSON is identical.
- Deliberate differences from v1: null-safe for empty repos (B20); the weekly heatmap always has 7 entries (v1 returned `[]` for an empty calendar).
- Tests: 1 parity + 13 engine + 14 archetype (all 5 archetypes, thresholds, priorities, infra-file signals).
- Bug log: B20 (v1 bug), B21, B22.

### 2026-10-13: PR #3 review: archetypes as D&D classes (🤖)
- Review: v1's archetypes weren't based on real habits, and the names should be fantasy (D&D) classes. Proposed and agreed: **code scores the class, the LLM adds the flavour** (Phase 5).
- 8 classes, `HabitProfile` (15 signals), `ScoringArchetypeClassifier` (0–100 per class, `ALGO_VERSION = 1`), `ArchetypeScores`; `InsightsResult(stats, habits, archetypeScores)`. `RuleBasedArchetypeClassifier` removed; the parity test now compares stats only.
- `V2__archetype_classes.sql`, so the developer's activity migration becomes `V3__activity.sql`.
- Calibration on tushar27x: Monk 52, Rogue 47, Paladin 44, Barbarian 44, Wizard 33, Artificer 33, Bard 10, Cleric 0.
- Tests: 32 habit-profile + 15 scoring + 14 engine + 1 parity, plus an opt-in live scoring test.
- Bug log: B23, B24.

### 2026-10-13: Phase 5 built (🤖)
- Spring AI 2.0's OpenAI module wraps the official `openai-java` SDK: a second model is built from `OpenAiChatOptions` (`baseUrl`, `apiKey`, `model`, `maxRetries`, `timeout`); `.options(...)` takes a builder. The primary is set to fail fast (1 retry, 30 s instead of 3 × 60 s) so the fallback gets its turn.
- StringTemplate renders a `List` with no separator (`[1, 2, 3]` → `123`), so lists are formatted in Java.
- Live: Groq had retired both v1 models (404). **v1 production is affected** (B26). Replacement gpt-oss models spent the whole `max_tokens` on reasoning until `reasoning_effort: low` + headroom (B27). The LLM gendered a user until the persona said not to (B28).
- Tests: 9 chain + 14 reviewer + 4 parallel-service (with a recording `FakeChatModel`, so the real `ChatClient` path runs), plus opt-in `AiLiveTest` (needs `GROQ_API_KEY`).
- Bug log: B25–B28.
- Developer's review: drop the roast. The habit tracker needs encouragement, so the persona became a supportive senior engineer and the roast became a year in review. Checked live: each review names real strengths and ends with one concrete step.
- PR #4 merged (`ce232bd`). Phase 5 follow-ups are listed in the Phase 7 hints.
- Open questions for the developer:
  - Archetype calibration: should Monk need small PRs as well as descriptive messages? Should zero activity stay Rogue?
  - v1 production still calls the retired Groq models (B26). Fix v1 (two model names in `ai_service.py`), or leave it until v2 replaces it?
