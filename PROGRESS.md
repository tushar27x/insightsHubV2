# Insight Hub v2: Progress & Roadmap

Rewrite of the Insight Hub backend (v1: Python/FastAPI in `~/dev/insight_hub/server`) in **Spring Boot 4 + Spring AI**, done as a learning project.
The goal is to understand the concepts, not just to reach feature parity. Each phase lists the concepts to learn, the v1 file to look at, hints, and a "done when" checklist.

**Status legend:** ⬜ not started · 🟨 in progress · ✅ done

---

## Current status

| Phase | Title | Status |
|------:|-------|:------:|
| 0 | Setup, infra & config | 🟨 |
| 1 | First endpoint: health ping | ⬜ |
| 2 | Persistence with JPA | ⬜ |
| 3 | GitHub GraphQL client | ⬜ |
| 4 | Insights engine (pure Java + first tests) | ⬜ |
| 5 | AI layer with Spring AI | ⬜ |
| 6 | Auth: GitHub OAuth + JWT (Spring Security) | ⬜ |
| 7 | Sync orchestration & Redis caching | ⬜ |
| 8 | Rate limiting | ⬜ |
| 9 | Errors, validation, CORS, logging | ⬜ |
| 10 | Frontend cutover & deployment | ⬜ |
| 11 | v2-only features (pgvector RAG, email digest) | ⬜ |

**Next up:** Finish Phase 0 (see the 2026-09-27 review in the learning log)

---

## v1 → v2 mapping (reference)

| v1 (Python) | What it does | v2 (Spring) equivalent |
|---|---|---|
| `app/main.py` | App startup, CORS, lifespan | `InsightshubApplication`, `@Configuration` classes |
| `app/core/config.py` | env vars via dotenv | `application.yml` + `@ConfigurationProperties` |
| `app/core/security.py` | JWT + Fernet encryption | Spring Security + `JwtEncoder`/`JwtDecoder`, AES-GCM (`javax.crypto`) |
| `app/db/postgres.py` | async SQLAlchemy engine | Spring Data JPA + HikariCP (auto-configured) |
| `app/db/redis.py` | redis client | `StringRedisTemplate` / Spring Cache |
| `app/models/user.py` | `UserInsights`, `UserTemplates` | `@Entity` classes + `JpaRepository` |
| `app/services/github_service.py` | GraphQL query to GitHub | `RestClient` + `.graphql` resource file |
| `app/services/insights_service.py` | stats + archetype computation | Plain `@Service` + Java records + streams |
| `app/services/ai_service.py` | Groq + Gemini fallback, persona | Spring AI `ChatClient`, prompt templates |
| `app/api/routes.py` | `/api/auth/*`, `/api/user/insights` | `@RestController`s + a `SyncService` |
| `fastapi-limiter` | rate limit per route | `HandlerInterceptor` + Redis (or Bucket4j) |

### API contract to preserve (the Next.js client depends on it)
- `GET /api/auth/login`: redirects to GitHub (sets `oauth_state` cookie)
- `GET /api/auth/callback?code&state`: syncs data, redirects to `FRONTEND_URL/dashboard?token=<jwt>`
- `GET /api/auth/logout`: redirects to `FRONTEND_URL`
- `GET /api/user/insights`: `Authorization: Bearer <jwt>`, returns **snake_case** JSON:
  `user_name, archetype, stats{...}, display_json, weekly_review, craft_review, updated_at`

---

## Phase 0: Setup, infra & config 🟨

**Concepts:** Spring Boot auto-configuration, starters, `application.yml`, profiles, externalized config, Docker Compose.

**Hints**
1. ⚠️ **Version check first.** Your `pom.xml` uses Spring Boot **4.1.1** with Spring AI **1.0.0**. Spring AI 1.0.x was built against Boot 3.x (Spring Framework 6, Jackson 2). Boot 4 moved to Framework 7 and Jackson 3. Find out which Spring AI line supports Boot 4 and align the versions. Otherwise you'll hit confusing auto-config errors later.
2. The starters you added will try to connect to Postgres, Redis and OpenAI on startup. Write a `compose.yaml` with Postgres (use the `pgvector/pgvector` image, since you'll want it in Phase 11) and Redis.
   Look up `spring-boot-docker-compose`, a dev-time dependency that starts `compose.yaml` automatically when you run the app.
3. Rename `application.yml` → `application.yml`. Read secrets from env vars with `${GITHUB_CLIENT_ID}` placeholders. Never commit secrets. Add a git-ignored `.env` or `application-local.yml`.
4. Create a typed config class: a Java `record` annotated with `@ConfigurationProperties(prefix = "insighthub")` holding `frontendUrl`, `backendUrl`, `jwtSecret`, `encryptionKey`, etc. Figure out how to enable it (`@ConfigurationPropertiesScan`).
5. Run `git init`. You'll want history for this.

**Done when**
- [x] Spring Boot / Spring AI versions are compatible
- [ ] Postgres (Neon) + Redis (remote) reachable from the app
- [ ] `./mvnw spring-boot:run` starts cleanly
- [ ] Config values come from env vars through a typed `@ConfigurationProperties` record

---

## Phase 1: First endpoint: health ping ⬜

**v1 reference:** `routes.py` → `GET /api/` (sets and gets a Redis key, runs `SELECT 1`)

**Concepts:** `@RestController`, `@GetMapping`, constructor injection, beans, `JdbcTemplate`, `StringRedisTemplate`, Actuator.

**Hints**
1. Unlike FastAPI's `Depends(get_db)`, Spring injects dependencies through the constructor. Use `final` fields + one constructor (or Lombok `@RequiredArgsConstructor`).
2. Return a `Map` or a `record` from the controller. Jackson serializes it for you.
3. Set a global `/api` prefix instead of repeating it on every controller (`server.servlet.context-path`, or a path prefix config). Think about which one also affects Actuator URLs.
4. Afterwards, look at `/actuator/health`. It already checks DB and Redis. That's the "Spring way" to do what v1's ping did.

**Done when**
- [ ] `GET /api/` returns `{"redis":"pong","postgres":1}`
- [ ] You can explain what a bean is and who creates your controller

---

## Phase 2: Persistence with JPA ⬜

**v1 reference:** `models/user.py`, `db/postgres.py`

**Concepts:** `@Entity`, `@Id`, `@OneToOne` + `@MapsId`, `JpaRepository`, JSONB columns, `ddl-auto` vs migrations, `@Transactional`.

**Hints**
1. `UserInsights.userId` is the GitHub ID, not generated. So there's no `@GeneratedValue`. Use `Long`.
2. `UserTemplates` shares its primary key with `UserInsights`. Look up `@MapsId`.
3. JSON columns: Hibernate 6+/7 supports `@JdbcTypeCode(SqlTypes.JSON)` on a `Map<String,Object>` field, or on a typed record. Use `jsonb` in Postgres.
4. Start with `spring.jpa.hibernate.ddl-auto=update` to move fast. Then consider **Flyway** (add the dependency) for proper versioned migrations, which is what you'd do in a real job.
5. Timestamps: prefer `Instant` or `OffsetDateTime` over `LocalDateTime`. This avoids v1's `tzinfo=None` juggling.
6. Decide: will v2 share the v1 Neon DB or use a fresh one? It matters for column names and for token encryption (Phase 6).

**Done when**
- [ ] Both tables are created from your entities (or migrations)
- [ ] `UserInsightsRepository.findById(...)` works
- [ ] You understand the owning side of the one-to-one relationship

---

## Phase 3: GitHub GraphQL client ⬜

**v1 reference:** `services/github_service.py`

**Concepts:** `RestClient` (the synchronous HTTP client that replaced `RestTemplate`), resources on the classpath, Jackson deserialization into records, error handling.

**Hints**
1. Put the query in `src/main/resources/graphql/user-insights.graphql` and load it with `@Value("classpath:...") Resource`.
2. Build one `RestClient` bean with `baseUrl("https://api.github.com")` using `RestClient.Builder` (auto-configured, so inject it).
3. Big design choice: map the response into **nested Java records** (type-safe, more code) or into a `JsonNode` (quick, like Python dicts). Recommendation: use records. That's where Java pays off. Use `@JsonIgnoreProperties(ignoreUnknown = true)`.
4. GraphQL returns HTTP 200 even on errors. Check the `errors` field yourself and throw a custom exception.
5. Test it manually with a personal access token before wiring OAuth (e.g. a temporary dev endpoint or a `CommandLineRunner`).

**Done when**
- [ ] Your GitHub profile is fetched and mapped into a record tree
- [ ] GraphQL errors raise a meaningful exception

---

## Phase 4: Insights engine (pure Java + first tests) ⬜

**v1 reference:** `services/insights_service.py` (`calculate_user_insights`, `determine_archetype`)

**Concepts:** Streams (`map`, `flatMap`, `sorted`, `Collectors.groupingBy`/`counting`), `LocalDate`/`DayOfWeek`, records, enums, **JUnit 5 + AssertJ** unit tests.

**Hints**
1. This is pure logic with no Spring needed. Keep it that way: input is the GitHub record from Phase 3, output is an `InsightsResult(Stats stats, Archetype archetype)` record.
2. Model the archetype as an `enum` with a display label (`"🕵️ The Bug Hunter"`).
3. The "flatten weeks → days, sort by date, take last N" logic appears 3 times in v1. Write it once.
4. Language counting: `flatMap` over repos → languages, then `groupingBy(identity(), counting())`, then sort by value.
5. Write unit tests **here first**. There's no DB and no HTTP, so it's a perfect TDD target. Cover each archetype branch.
6. Note: the generated `InsightshubApplicationTests.contextLoads()` starts the full context, so it needs DB/Redis/OpenAI. Decide how to handle it (Phase 10: Testcontainers).

**Done when**
- [ ] Output matches v1's `stats` JSON shape for the same input
- [ ] Unit tests cover all 5 archetypes + heatmap edge cases (< 7 days)

---

## Phase 5: AI layer with Spring AI ⭐ ⬜

**v1 reference:** `services/ai_service.py`

**Concepts:** `ChatClient` fluent API, system vs user messages, `PromptTemplate` (StringTemplate `.st` files), OpenAI-compatible providers, multiple model beans + `@Qualifier`, structured output (`.entity(...)`), fallback strategies, parallelism with `CompletableFuture` / virtual threads.

**Hints**
1. **You don't need a Groq SDK.** Groq exposes an OpenAI-compatible API. Point the OpenAI starter's `base-url` at Groq and set the model to `llama-3.3-70b-versatile`. Gemini also has an OpenAI-compatible endpoint, which is useful for the fallback.
2. Put `MANAGER_PERSONA` in `resources/prompts/persona.st` and each review prompt in its own `.st` file with `{variables}`. Load them as `Resource`s.
3. Build a `ChatClient` bean with `.defaultSystem(persona)` so every call carries the persona.
4. Fallback: you'll need **two** chat clients (primary + fallback). Auto-config gives you one, so read up on creating a second `OpenAiChatModel` manually and telling them apart with `@Qualifier`. Wrap the call in try/catch → fallback → static message, the same as v1. (Stretch: Spring Retry or Resilience4j.)
5. `generate_repo_summaries` returns a `Map<String,String>`. Try `.call().entity(new ParameterizedTypeReference<...>())` to get structured output in **one** call instead of 3.
6. v1 used `asyncio.gather` for 4 AI calls in parallel. In Java: `CompletableFuture.supplyAsync(..., executor)` + `allOf`. Enable virtual threads (`spring.threads.virtual.enabled=true`) and use a virtual-thread executor. Handle each future's exception separately (like `return_exceptions=True`).
7. Per-call options (`max_tokens`, model override for the cheaper 8B model) go through `.options(...)`.

**Done when**
- [ ] All 4 generators work against Groq
- [ ] Killing the Groq key falls back to Gemini (the Java version of v1's `test_ai.py`)
- [ ] The 4 calls run in parallel (measure the time)

---

## Phase 6: Auth: GitHub OAuth + JWT (Spring Security) ⬜

**v1 reference:** `routes.py` (`/auth/login`, `/auth/callback`, `/auth/logout`), `core/security.py`

**Concepts:** Spring Security filter chain, `SecurityFilterChain` bean, stateless sessions, OAuth2 authorization code flow, JWT (Nimbus: `JwtEncoder`/`JwtDecoder`), `@AuthenticationPrincipal`, symmetric encryption.

**Hints**
1. You need new dependencies: `spring-boot-starter-security`, and `spring-boot-starter-oauth2-resource-server` for JWT validation (it brings Nimbus).
2. **Two approaches for the GitHub login part:**
   - **(a) Hand-rolled, like v1:** controller builds the authorize URL, stores `state` in a cookie, and the callback exchanges the code with `RestClient`. Easiest to reason about.
   - **(b) `spring-boot-starter-oauth2-client`:** Spring handles state/redirect/code exchange, and you plug in an `AuthenticationSuccessHandler` that issues your JWT and redirects.
   - Recommendation: start with **(a)** to understand the flow, then refactor to **(b)** as a learning exercise.
3. JWT issuing: define `JwtEncoder`/`JwtDecoder` beans using an HMAC secret (`NimbusJwtEncoder` + `ImmutableSecret`). Claims: `sub=githubId`, `name=login`, `exp=24h`.
4. Protect `/user/**`, permit `/auth/**` and health. Use `SessionCreationPolicy.STATELESS` and disable CSRF (Bearer tokens only).
5. Encrypting the GitHub token: Fernet is Python-specific. Use **AES-GCM** via `javax.crypto` (random 12-byte IV prepended, Base64 encoded). If you share v1's DB, existing encrypted tokens won't decrypt, so plan for that. (Note: v1 declares the column but never actually stores the token.)
6. Read the user in controllers with `@AuthenticationPrincipal Jwt jwt` → `jwt.getSubject()`.

**Done when**
- [ ] Full login round-trip lands on the frontend with `?token=`
- [ ] `GET /user/insights` returns 401 without a token and 200 with one

---

## Phase 7: Sync orchestration & Redis caching ⬜

**v1 reference:** the big body of `callback()` + `get_user_insights()` in `routes.py`

**Concepts:** Service layer design, `@Transactional` boundaries, Spring Cache abstraction (`@EnableCaching`, `@Cacheable`, `@CacheEvict`), `RedisCacheManager` config, TTLs, JSON serialization in Redis.

**Hints**
1. The v1 callback does far too much. Extract a `SyncService.syncIfStale(githubId, login, githubToken)`: check staleness (7 days) → fetch → compute → AI → upsert → evict cache. Keep controllers thin.
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

## Phase 9: Errors, validation, CORS, logging ⬜

**Concepts:** `@RestControllerAdvice`, `ProblemDetail` (RFC 9457), custom exceptions, Bean Validation (`@Valid`, `@NotBlank`), CORS in Spring Security, SLF4J (`@Slf4j`), log levels per package.

**Hints**
1. Replace every `print(...)` from v1 with proper log levels. **Never log tokens** (v1 printed the GitHub token response!).
2. Map exceptions to responses in one place: `GitHubApiException` → 502, `UserNotFoundException` → 404, and so on.
3. CORS must be configured through Spring Security's `.cors(...)` + a `CorsConfigurationSource` bean, or the security filter chain will block preflight requests.
4. Validate `code`/`state` params on the callback.

**Done when**
- [ ] All errors return consistent `ProblemDetail` JSON
- [ ] No secrets in logs

---

## Phase 10: Frontend cutover, testing & deployment ⬜

**Concepts:** Jackson naming strategies, integration tests with `@SpringBootTest` + **Testcontainers**, `@WebMvcTest` slice tests, `MockMvc`, Docker images (`spring-boot:build-image` / Dockerfile), production profile.

**Hints**
1. The client expects snake_case JSON. Either set a global `SNAKE_CASE` naming strategy or use `@JsonProperty` per field. Know the trade-off.
2. Point `NEXT_PUBLIC_API_URL` at `http://localhost:8080/api` and run the v1 frontend unchanged.
3. Add Testcontainers (Postgres + Redis) and look up `@ServiceConnection`. Fix `contextLoads()`.
4. `@WebMvcTest` your controllers with a mocked service. Test the security rules.
5. Build an image: `./mvnw spring-boot:build-image`. Create an `application-prod.yml`.

**Done when**
- [ ] The v1 Next.js frontend works end to end against the Java backend
- [ ] `./mvnw verify` is green with integration tests

---

## Phase 11: v2-only features ⬜ *(proposals: confirm scope before starting)*

You added `pgvector`, `mail` and `actuator`, which v1 never used. Ideas that fit them:

**11a. "Ask your Manager" (RAG chat)**
- Embed each user's README texts, commit messages and repo summaries into the pgvector `VectorStore` (with `userId` metadata) during sync.
- New endpoint `POST /user/ask {question}` → `ChatClient` + `QuestionAnswerAdvisor` (filter by `userId`), answering in the persona.
- Concepts: embeddings, `Document`, `VectorStore`, metadata filters, advisors, chat memory. Note: Groq has no embeddings API, so you'll need a separate embedding model.

**11b. Weekly performance email**
- `@Scheduled` job (cron: Mondays) → re-sync stale users → `JavaMailSender` HTML email with the weekly review.
- Concepts: `@EnableScheduling`, cron expressions, Thymeleaf templates for email, opt-in flag on the user.

**11c. Observability**
- Actuator + Micrometer: count AI calls, fallback rate, latency per model. Spring AI emits observations out of the box.

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
  - `insightshub:` is nested under `spring:` in the yml, so the actual prefix is `spring.insightshub`.
  - `InsightsHubProperties` has package-private fields with no setters or constructor, so nothing binds (use a `record`).
  - ~~Spring Boot doesn't read `.env`~~ → added `spring-dotenv` 4.0.0.
  - ~~`.env` not in `.gitignore`~~ → fixed. Still need `git init`.
  - No `spring.ai.openai.*` config. The OpenAI auto-config will likely fail at startup.
- Decision: use the production Neon DB + remote Redis for dev. Only the owner's data exists, so `ddl-auto: update` is acceptable.
