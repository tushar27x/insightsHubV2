# Insight Hub v2: Progress & Roadmap

Rewrite of the Insight Hub backend (v1: Python/FastAPI in `~/dev/insight_hub/server`, repo `tushar27x/insight_hub`) in **Spring Boot 4 + Spring AI** (repo `tushar27x/insightsHubV2`), done as a learning project.
The goal is to understand the concepts, not just to reach feature parity. Each phase lists the concepts to learn, the v1 file to look at (if any), hints, and a "done when" checklist.

**Status legend:** ⬜ not started · 🟨 in progress · ✅ done

---

## Scope & decisions

**Why v2:** a Spring Boot project for the resume, with Spring AI replacing v1's Python LLM calls. v2 ports v1, then adds new features on top.

**What v2 adds beyond the port**
1. **Habit tracker:** a daily scheduled sync of each user's GitHub activity, kept as history.
2. **Alerts:** detected habits are sent to the user in a **daily digest email**.
3. **Archetype ranking:** archetypes become scores, and users are ranked within their archetype, globally.

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

---

## Current status

| Phase | Title | Status |
|------:|-------|:------:|
| | **Part A: Port v1** | |
| 0 | Setup, infra & config | ✅ |
| 1 | First endpoint: health ping | ✅ |
| 2 | Data model for v2 (JPA) | 🟨 |
| 3 | GitHub GraphQL client | ⬜ |
| 4 | Insights engine (pure Java + first tests) | ⬜ |
| 5 | AI layer with Spring AI | ⬜ |
| 6 | Auth: GitHub OAuth + JWT, stored token & email | ⬜ |
| 7 | Sync orchestration & Redis caching | ⬜ |
| 8 | Rate limiting | ⬜ |
| 9 | Errors, validation, logging | ⬜ |
| | **Part B: New v2 features** | |
| 10 | Scheduled daily sync & activity history | ⬜ |
| 11 | Habit engine (3 rules) | ⬜ |
| 12 | Notifications: daily digest email | ⬜ |
| 13 | Archetype scoring & leaderboards | ⬜ |
| | **Part C: Ship it** | |
| 14 | Testing, API docs & deployment | ⬜ |
| | **Stretch** | |
| S | RAG chat, Discord alerts, friends leaderboard, observability | ⬜ |

**Next up:** Phase 2 step 6: `V2__activity.sql` (`daily_contributions` composite key, `commits` unique `sha`)

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
| `insights` | Latest computed stats JSON, AI texts (roast, weekly, craft), archetype | Replaces v1's `user_templates` |
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

## Phase 3: GitHub GraphQL client ⬜

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
- [ ] Your GitHub profile is fetched and mapped into a record tree, including commit SHAs and dates
- [ ] GraphQL errors raise a meaningful exception
- [ ] `rateLimit.cost` / `remaining` show up in the logs

---

## Phase 4: Insights engine (pure Java + first tests) ⬜

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
- [ ] Output matches v1's `stats` JSON shape for the same input
- [ ] Unit tests cover all 5 archetypes + heatmap edge cases (< 7 days)

---

## Phase 5: AI layer with Spring AI ⭐ ⬜

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
8. Design the AI service so Phase 12 can reuse it. The digest email's intro will be written in the same manager persona.

**Done when**
- [ ] All 4 generators work against Groq
- [ ] Killing the Groq key falls back to Gemini (the Java version of v1's `test_ai.py`)
- [ ] The 4 calls run in parallel (measure the time)

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
4. `@CacheEvict(value="user_insights", key="#userId")` on the sync's persist step replaces v1's manual `redis.delete`.
5. v1 swallows sync errors so login still works. Keep that behavior, but log properly.

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

**Goal:** once a day, each user with pending alerts gets **one** email that summarises them in the manager persona.

**Concepts:** `JavaMailSender` + `MimeMessageHelper`, HTML email templates (Thymeleaf), Strategy pattern (`NotificationChannel`), delivery guarantees (at-least-once vs at-most-once), signed unsubscribe links, a preferences API.

**Hints**
1. `interface NotificationChannel { void send(User user, Digest digest); }` with an `EmailChannel` implementation. Discord comes later as a second implementation (stretch).
2. Digest job (`@Scheduled`, after the sync job): for each user with pending `habit_alerts` (`digest_id IS NULL`) → build one digest → send → mark the alerts as sent. **No alerts, no email.**
3. Think through failure: if sending succeeds but marking fails, the user gets a duplicate tomorrow. If you mark first and sending fails, the alert is lost. Pick one and write down why (a digest that occasionally repeats is usually the lesser evil).
4. Content: Spring AI writes a short intro in the manager persona from the alert list (reuse Phase 5). **The email must still go out if the AI call fails**, so have a plain template fallback.
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

**Concepts:** normalisation and scoring design, versioned algorithms, SQL window functions (`RANK() OVER (PARTITION BY ...)`, supported in Hibernate 6+ HQL), Spring Data `Pageable`, Redis **sorted sets** (`ZADD`, `ZREVRANK`, `ZREVRANGE`), profiles for seed data.

**Hints**
1. Replace the first-match rules (Phase 4) with a scorer: compute a 0–100 score for **each** archetype from normalised metrics (e.g. issues ratio → Bug Hunter, PR ratio → Open Source Hero, infra files → Architect, weekend share → Weekend Warrior). Primary archetype = highest score, with a fixed tie-break order.
2. Store the breakdown (which metric contributed what) as JSON, so you can explain a score, and maybe let the AI roast it.
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

Add an entry at the end of each session: what was built, what was learned, open questions.

### 2026-09-27: Kickoff
- Scanned the v1 backend (`~/dev/insight_hub/server`) and the v2 skeleton.
- Created the roadmap (this file) and `.claude/CLAUDE.md`.
- Open question: Spring AI version vs Boot 4 compatibility (Phase 0, hint 1).
- Open question: share the v1 database or start fresh?

### 2026-09-27: Phase 0 review
- ✅ Bumped Spring AI to 2.0.0 (Boot 4 compatible). `./mvnw compile` passes.
- ✅ `application.yml` with `${ENV}` placeholders, `@ConfigurationPropertiesScan` enabled, `@Validated` properties class.
- ✅ Chose remote Neon Postgres + remote Redis (TLS) over local Docker Compose.
- ❌ Blockers found:
  - ~~`insightshub:` nested under `spring:`~~ → fixed, now top level.
  - ~~`InsightsHubProperties` can't bind~~ → converted to a record.
  - ~~`.env` not loaded~~: `spring-dotenv` 4.0.0 (2023) is built for Boot 3. In Boot 4, `ConfigurableBootstrapContext` moved to `org.springframework.boot.bootstrap`, so the library's run listener never fires. The DB login used the literal `${DB_USER}`. Fix: built-in `spring.config.import: optional:file:.env[.properties]`.
  - ✅ Removed `spring-dotenv`, added `spring.config.import: optional:file:.env[.properties]`, restored `.env`. DB connects (Neon, Postgres 17.11).
  - ~~`.env` not in `.gitignore`~~ → fixed. Repo initialized; first commit `1e5306a`, `.env` not tracked.
  - ✅ ~~OpenAI auto-config~~ → Groq key + model configured; unused models switched off with `spring.ai.model.{audio.speech,audio.transcription,image,moderation}: none`. Was: `OpenAiAudioSpeechAutoConfiguration` fails with "At least one credential source must be specified". The OpenAI starter auto-configures chat, embedding, image, audio speech, audio transcription and moderation models, and each needs a key. Fix: Groq key + base URL, and turn off unused models with `spring.ai.model.<type>=none` (and `spring.ai.vectorstore.type=none` until Phase 11).
- Decision: use the production Neon DB + remote Redis for dev. Only the owner's data exists, so `ddl-auto: update` is acceptable.
- ✅ `@Validated` caught 4 blank `insightshub.*` values at startup (empty in `.env`), so fail-fast validation works.

### 2026-09-27: Phase 0 complete ✅
- The app starts: Neon Postgres + remote Redis connect, `.env` loads through `spring.config.import`, and `InsightsHubProperties` binds and validates.
- Spring AI: Groq as the OpenAI-compatible chat provider. Speech, transcription, image and moderation are turned off with `spring.ai.model.*: none` (replaced an earlier `spring.autoconfigure.exclude` list, since the switches survive class renames).
- Learned: starters switch features on and properties or excludes switch them off; `@Validated` fails fast; Boot 3 libraries can silently break on Boot 4 (spring-dotenv); the real cause is at the first `WARN`/`Caused by`, not the last stack trace.
- ✅ ~~Carry-over to Phase 5: `base-url` must be `https://api.groq.com/openai/v1`~~ → fixed. Probed without a key: `/openai/v1/chat/completions` returns 401 (exists), while `/openai/chat/completions` and `/v1/openai/chat/completions` return 404.
- Embedding model + pgvector store turned off (`spring.ai.model.embedding: none`, `spring.ai.vectorstore.type: none`). Turn them back on for stretch goal S1 with a real embedding provider.
- 🐛 Found while turning off embeddings: `vectorstore.type: none` was indented under `spring.ai.model`, so it became `spring.ai.model.vectorstore.type`, a key nothing reads. With `embedding: none` in effect, pgvector still loaded and failed: "PgVectorStoreAutoConfiguration required a bean of type EmbeddingModel". The correct key is `spring.ai.vectorstore.type`. Lesson: an unknown key is silently ignored, not flagged.
- ✅ Fixed: `vectorstore` moved up to `spring.ai.vectorstore.type`. The app starts cleanly again. Committed as `89f50dc`.

### 2026-10-01: Scope change: v2 features defined
- v2 = port + **habit tracker** + **daily digest alerts** + **archetype ranking**. Backend-only; RAG becomes a stretch goal.
- Habits: streak broken, no commits for N days, lazy commit messages. Delivery: daily digest email.
- Roadmap restructured into Part A (port, phases 1–9), Part B (new features, phases 10–13), Part C (ship, phase 14) and stretch goals.
- Consequences for the port: Phase 2 now designs history and score tables up front; Phase 3 fetches commit SHAs/dates and logs `rateLimit`; Phase 6 adds the `user:email` scope and stores the GitHub token encrypted; Phase 7's `SyncService` must work without an HTTP request.
- ⚠️ v1 is still live on the same Neon DB, so v2 should use its own schema/tables.

### 2026-10-01: Phase 1 started
- Roadmap committed (`9ce8ed3`). Working tree clean.
- `HealthController` written (package `health`, Lombok `@RequiredArgsConstructor`, `StringRedisTemplate` + `JdbcTemplate`, record response) at `GET /ping`.
- 🐛 First request failed: `Cannot parse port number: https://pure-titmouse-69745.upstash.io"`. `REDIS_HOST` in `.env` held Upstash's **REST URL** (with `https://`) plus a stray trailing `"`. Spring Data Redis (Lettuce) speaks the Redis TCP protocol, so it needs the bare hostname (port 6379, TLS on).
- Lesson: the bad value had been there since Phase 0, but startup passed because **Redis connects lazily** (on the first command), while Hibernate connects to Postgres eagerly at startup to read DB metadata.
- ✅ Fixed `REDIS_HOST` (bare Upstash hostname). `GET /ping` → `{"redis":"pong","jdbc":1}`; `/actuator/health` → `UP` (details hidden by default).
- Q: "Doesn't Actuator already have a health check?" Yes. `/actuator/health` aggregates `HealthIndicator` beans (`db`, `redis`, `diskSpace`, `ping`...), auto-registered from the starters. A hand-rolled ping is only useful as a learning exercise or to demo a custom read/write round trip.
- Remaining for Phase 1: rename `jdbc` → `postgres`, make the response record public (or its own file), pick the `/api` prefix strategy, show health details.
- ✅ Enabled `management.endpoint.health.show-details: always`. `/actuator/health` → `db`, `redis`, `diskSpace`, `ping`, `ssl`, `livenessState`, `readinessState` all `UP`.
- Decision: deleted `HealthController`; Actuator is the health check. (Switch `show-details` to `when-authorized` in Phase 6.)
- Carried to Phase 6: the `/api` prefix decision (`server.servlet.context-path` vs a controller prefix).

### 2026-10-01: Phase 1 complete ✅ → Phase 2 started
- Schema review #1 (users ↔ insights sketch). Fixes: `user` is a reserved word in Postgres → `users`; flip the 1:1 so `insights.user_id` is PK + FK (`@MapsId`), not `users.insight_id`; don't store `global_rank` (it depends on other rows, so derive it in Phase 13); archetype as a `varchar` + CHECK on `insights` (computed data), not an FK on `users`; drop weekly/monthly (decision = daily digest); add `login`, `encrypted_github_token`, `needs_reauth`, `no_commit_days`, `public_profile`, timestamps; split `insights` JSON into `stats jsonb` plus text columns for the AI reviews.
- ✅ Flyway added (`spring-boot-starter-flyway` + `flyway-database-postgresql`; Boot 4 needs its own starter). Config: `spring.flyway.schemas/default-schema: v2`, `hibernate.default_schema: v2`, `ddl-auto: validate`.
- ✅ `V1__users_and_insights.sql` applied to Neon: created schema `v2`, `v2.flyway_schema_history`, `v2.users`, `v2.insights`. v1's `public` schema untouched. Worked through the Neon pooler endpoint.
- Archetype names are now fixed by the CHECK constraint: `BUG_HUNTER, OPEN_SOURCE_HERO, SOFTWARE_ARCHITECT, WEEKEND_WARRIOR, CODE_CRUSADER`. The Java enum (Phase 4) must use exactly these names.
- Next: `User` + `Insights` entities and repositories (Phase 2, step 4).
- Review of `enity/Users.java`: it compiles, but startup fails validation: `wrong column type ... [github_id]; found [int8 (BIGINT)], but expecting [numeric(38,0)]`. `BigInteger` maps to NUMERIC; BIGINT needs `Long`. Also: no no-arg constructor (JPA requires one), snake_case Java fields (break Spring Data derived queries, where `_` means nested property), `java.sql.Timestamp` → `Instant`, null `Boolean`s vs NOT NULL (a DB DEFAULT only applies when the column is omitted from the INSERT, and Hibernate always sends every column), `@Column(name="githubId")` only works because Boot's `CamelCaseToUnderscoresNamingStrategy` rewrites explicit names too, `enity` typo, class should be singular `User`, no getters.
- ✅ `Users` entity passes `ddl-auto: validate` (`Long` id, `Instant`, correct column names, protected no-arg constructor). App starts.
- Still open: `alerts_enabled` field still snake_case; `publicProfile`/`needsReauth` have no Java default, so an INSERT will send NULL and violate NOT NULL; no public constructor or getters, so app code can't create or read a user; `enity` → `entity`, `Users` → `User`.
- Review #3 of `entity/Users.java`: package renamed ✅, flags now primitives so the NOT NULL issue is fixed ✅, `@Getter` ✅. Open: no public `(githubId, login)` constructor, so app code can't create users; no setters on `encryptedGithubToken`/`needsReauth`/`publicProfile` (needed in Phases 6/10/12); class still `Users`. Rule noted: never return entities from controllers, since `@Getter` would leak the encrypted token through Jackson; use DTOs.
- Review #4: all setters ✅, `int noCommitDays` ✅. 🐛 `public void User(Long, String)` is a **method**, not a constructor: it has a return type and the class is still named `Users`. Java allows methods with a class-like name, so it compiles silently.
- ✅ `Users` entity done: real public `(githubId, login)` constructor, protected no-arg constructor for JPA, getters, setters on mutable fields, primitive flags, validates against `v2.users`. (Class kept as `Users`.)
- Next: `UserRepository`, `Archetype` enum, `Insights` entity with `@MapsId`.
- Review: `Archetype`, `Insights`, `UsersRepository`. Startup fails at repository creation: `No property 'existsUser' found for type 'Users'`. Derived queries must follow `existsBy<Property>`; `existsById` is inherited anyway, as is `findById`. Schema validation of both entities **passed** (the EntityManagerFactory is built before repositories). So Hibernate 7 accepts `text` columns for plain `String` fields, and the earlier "columnDefinition = text is needed" claim was wrong: it's optional, just documentation.
- Open in `Insights`: `stats` and `updatedAt` are null on a new object but NOT NULL in the DB; no constructor taking `Users`; no setters for `stats`/`archetype`; public no-arg constructor; `userId` not private; unused `java.sql.SQLType` import. `""` defaults hide "not generated yet". No `InsightsRepository` yet.
- `findByNeedsReauthFalseAndLastSyncedAtBefore` skips never-synced users (NULL < cutoff is not true in SQL). Adding `Or...IsNull` binds wrong (OR has lower precedence), so use `@Query` (JPQL) in Phase 10.
- ✅ Review fixes applied:
  - `UsersRepository`: removed `existsUser`/`findById`; replaced the sync query with `@Query findDueForSync(cutoff)` (`needsReauth = false and (lastSyncedAt is null or lastSyncedAt < :cutoff)`).
  - `Insights`: protected no-arg + `Insights(Users)` constructor, `stats = new HashMap<>()`, `@UpdateTimestamp updatedAt`, setters for stats/archetype/AI texts, AI texts null by default, `private userId`, `@Table`.
  - `Archetype`: `name` → `slug`.
  - Added `InsightsRepository`.
- ✅ The app starts: both entities validate, and the JPQL query parses at startup. Not yet verified: an actual save/read round trip (step 5).
- 🐛 IntelliJ showed `Cannot resolve symbol 'log'` / `getGithubId` while `./mvnw compile` passed. Cause: the Arch `intellij-idea-community-edition` package (build IC-262.10315.125) doesn't bundle the Lombok plugin. Annotation processing settings were already correct. Fixed by installing the official JetBrains Lombok plugin (262.8665.176, compatible with `262.*`) into `~/.local/share/JetBrains/IdeaIC2026.2/lombok/`. Needs an IDE restart.
- Lesson: IDE errors + green Maven build = IDE setup issue, not code.
- ✅ Round trip, run 1 (`dev` profile): user + insights saved to `v2`. `insights.userId = 100276134` without setting it (`@MapsId` works); `stats` jsonb round-tripped; archetype stored as `CODE_CRUSADER`.
- `createdAt=null` after `save()`: the DB filled it (`DEFAULT now()`), but Hibernate never reads back columns it didn't write (`insertable=false`).
- 💥 `LazyInitializationException: Could not initialize proxy [Users#100276134] - no session` on `loaded.getUser().getLogin()`. `findById` runs in its own transaction; `user` is a LAZY proxy, and the session closed when the repository call returned. Fixes: a transaction around the whole unit of work (service-layer `@Transactional`, Phase 7), a fetch join / `@EntityGraph`, or load the user via its own repository.
- Noted the startup WARN about `spring.jpa.open-in-view`: OSIV keeps the session open for the whole web request, which would *hide* this exact bug in controllers. Plan: set it to `false`.
- Changes: `@Generated(event = INSERT)` on `Users.createdAt`; `@EntityGraph(attributePaths = "user")` override of `InsightsRepository.findById`.
- Run 2 (data from run 1 still present):
  - `usersRepository.save(new Users(id, login))` did **not** fail. The id is non-null, so Spring Data treats the entity as existing and calls `merge` (SELECT + UPDATE). ⚠️ The merge copies *every* field from the new object, so it silently resets the existing user's preferences (`alertsEnabled`, `noCommitDays`, `publicProfile`...) to defaults. Rule for Phase 6: for existing users, load and then update. Never `save(new Users(...))`.
  - `createdAt` still null: it was an UPDATE, so the INSERT-generated value isn't re-read, and merge copied our null onto the loaded object.
  - `insightsRepository.save(new Insights(user))` → `duplicate key ... insights_pkey`. Its `@Id` is null before `@MapsId` fills it, so `isNew()` is true → `persist` → INSERT → conflict.
  - The `@EntityGraph` line wasn't reached, so it's not verified yet.
- ✅ `spring.jpa.open-in-view: false` (startup warning gone).
- Runner: user is now find-or-create → `createdAt=2026-10-06T03:57:12Z` (loaded from the DB). Insights is still `new Insights(user)` every run → same `insights_pkey` duplicate. Needs find-or-create too (`findById(id).orElseGet(() -> new Insights(user))`, then set fields and save, which becomes an UPDATE for an existing row).
- ✅ Step 5 done. The runner is idempotent (find-or-create for both user and insights). Run 3: no errors, `createdAt` populated, `Insights belong to tushar27x` (the `@EntityGraph` fetch removes the LazyInitializationException). Runner kept behind `@Profile("dev")`.
