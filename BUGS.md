# Bug Log

Every bug hit while building Insight Hub v2: what we saw, why it happened, how it was fixed, and what to remember.
New bugs get the next `B` number. PR descriptions list the bugs found while building that phase.

| # | Phase | Bug | Kind |
|---|---|---|---|
| [B1](#b1) | 0 | Spring AI 1.0.0 doesn't support Spring Boot 4 | Dependency |
| [B2](#b2) | 0 | `insightshub:` indented under `spring:`, so properties bound to the wrong prefix | Config |
| [B3](#b3) | 0 | `@ConfigurationProperties` class with no setters/constructor bound nothing | Code |
| [B4](#b4) | 0 | `spring-dotenv` silently did nothing on Boot 4 | Dependency |
| [B5](#b5) | 0 | `.env` not git-ignored (caught before the first commit) | Security |
| [B6](#b6) | 0 | OpenAI starter failed on audio speech: "At least one credential source must be specified" | Config |
| [B7](#b7) | 0 | `vectorstore` nested under `spring.ai.model`, so it was ignored and pgvector demanded an `EmbeddingModel` | Config |
| [B8](#b8) | 0 | Groq `base-url` missing `/v1`, then with segments reversed | Config |
| [B9](#b9) | 1 | `REDIS_HOST` held a REST URL + stray quote: "Cannot parse port number" | Config |
| [B10](#b10) | 2 | `BigInteger` id failed schema validation against `BIGINT` | Code |
| [B11](#b11) | 2 | Entity defaults: missing no-arg constructor, null flags vs `NOT NULL` columns | Code |
| [B12](#b12) | 2 | `public void User(...)` was a method, not a constructor | Code |
| [B13](#b13) | 2 | Invalid derived query `existsUser` stopped startup | Code |
| [B14](#b14) | 2 | Sync query would skip never-synced users (`NULL < cutoff`) | Logic |
| [B15](#b15) | 2 | IntelliJ "Cannot resolve symbol 'log'" while Maven compiled fine | Tooling |
| [B16](#b16) | 2 | `createdAt` null right after `save()` | Code |
| [B17](#b17) | 2 | `LazyInitializationException ... no session` | Code |
| [B18](#b18) | 2 | `save(new Entity(existingId))`: silent overwrite for `Users`, duplicate key for `Insights` | Code |
| [B19](#b19) | 3 | `RestClient.Builder` auto-config only present via the Spring AI starter | Dependency |
| [B20](#b20) | 4 | **v1:** insights crash on any empty repository (`defaultBranchRef: null`) | Logic (v1) |
| [B21](#b21) | 4 | Phase 4 branch created from a stale local `main` | Process |
| [B22](#b22) | 4 | `reference to assertThat is ambiguous` with Jackson's `valueToTree` | Test code |

---

### B1
**Spring AI 1.0.0 doesn't support Spring Boot 4** · Phase 0 · Dependency
- **Symptom:** caught during planning, before it could break anything. Spring AI 1.0.x targets Boot 3 (Spring Framework 6, Jackson 2); the project uses Boot 4 (Framework 7, Jackson 3).
- **Fix:** `spring-ai.version` → `2.0.0`.
- **Lesson:** after a major Boot upgrade, check every Spring-ecosystem library's supported Boot line first.

### B2
**Properties bound to `spring.insightshub` instead of `insightshub`** · Phase 0 · Config
- **Symptom:** `@ConfigurationProperties(prefix = "insightshub")` received nothing.
- **Root cause:** in `application.yml`, the `insightshub:` block was indented under `spring:`.
- **Fix:** made `insightshub:` a top-level key.
- **Lesson:** YAML nesting *is* the property name. Spring ignores keys it doesn't recognize, so check indentation first.

### B3
**`InsightsHubProperties` couldn't bind** · Phase 0 · Code
- **Root cause:** a plain class with package-private fields and no setters or constructor, so Spring had no way to set the values.
- **Fix:** turned it into a `record` (constructor binding).
- **Lesson:** records are the simplest immutable way to bind configuration.

### B4
**`.env` not loaded: DB login as the literal user `${DB_USER}`** · Phase 0 · Dependency
- **Symptom:** `password authentication failed for user '${DB_USER}'`, shown at the bottom as `Unable to determine Dialect without JDBC metadata`.
- **Root cause:** `spring-dotenv` 4.0.0 (2023) was built against Boot 3. Boot 4 moved `ConfigurableBootstrapContext` to `org.springframework.boot.bootstrap`, so the library's listener method no longer overrode anything and never ran. No error was raised.
- **Fix:** removed the library; `spring.config.import: "optional:file:.env[.properties]"`.
- **Lesson:** the real cause is at the first `WARN`/`Caused by`, not the last stack trace. Prefer built-in features over small libraries across major upgrades.

### B5
**`.env` not git-ignored** · Phase 0 · Security
- **Symptom:** found in review before `git init`.
- **Fix:** rewrote `.gitignore` with secrets first (`.env`, `.env.*`, keys, local profiles); verified `.env` was never committed.
- **Lesson:** set up `.gitignore` before the first `git add .`, especially for a public repo.

### B6
**`OpenAiAudioSpeechAutoConfiguration`: "At least one credential source must be specified"** · Phase 0 · Config
- **Root cause:** the OpenAI starter auto-configures chat, embedding, image, audio speech, transcription and moderation models, each needing a key. None was configured.
- **Fix:** Groq API key + base URL for chat; `spring.ai.model.{audio.speech,audio.transcription,image,moderation,embedding}: none`. (First fixed with `spring.autoconfigure.exclude`, then switched to these properties, which survive class renames.)
- **Lesson:** a starter switches features on; properties or excludes switch them off.

### B7
**pgvector: "required a bean of type `EmbeddingModel` that could not be found"** · Phase 0 · Config
- **Root cause:** `vectorstore: type: none` was nested under `spring.ai.model`, so the real key `spring.ai.vectorstore.type` was never set. With embeddings disabled, pgvector still loaded and had no embedding model.
- **Fix:** moved it to `spring.ai.vectorstore.type: none`.
- **Lesson:** same as B2: an unknown key is ignored silently, and the symptom shows up somewhere else.

### B8
**Groq base URL wrong** · Phase 0 · Config
- **Symptom:** no error at startup (no API call is made); found by probing the endpoints.
- **Root cause:** Spring AI 2.0's default base URL includes `/v1` (`https://api.openai.com/v1`). `https://api.groq.com/openai` and `https://api.groq.com/v1/openai` both return 404.
- **Fix:** `https://api.groq.com/openai/v1` (returns 401 without a key, i.e. the endpoint exists).
- **Lesson:** "the app starts" doesn't mean an external integration works.

### B9
**`Cannot parse port number: https://<host>.upstash.io"`** · Phase 1 · Config
- **Symptom:** the first request using Redis returned 500.
- **Root cause:** `REDIS_HOST` held Upstash's REST URL (`https://...`) plus a stray `"`. Lettuce speaks the Redis protocol and needs a bare hostname.
- **Fix:** bare hostname in `.env`; port 6379 and TLS stay in `application.yml`.
- **Lesson:** Redis connects **lazily** (first command); Postgres connects **eagerly** at startup. The bad value had been there since Phase 0.

### B10
**`wrong column type ... [github_id]; found [int8], but expecting [numeric(38,0)]`** · Phase 2 · Code
- **Root cause:** `BigInteger` maps to SQL `NUMERIC`.
- **Fix:** `Long` for `BIGINT`.
- **Lesson:** `ddl-auto: validate` catches type mismatches at startup, so keep it on.

### B11
**Entity construction and defaults** · Phase 2 · Code (found in review)
- **Problems:** `@AllArgsConstructor` removed the no-arg constructor JPA needs; `Boolean` fields defaulted to `null` against `NOT NULL` columns; same for `Insights.stats` and `updatedAt`.
- **Root cause:** a DB `DEFAULT` only applies when the INSERT *leaves the column out*. Hibernate sends every column, so `null` is sent.
- **Fix:** protected no-arg constructor + public constructor for the required fields; primitive flags with Java defaults; `stats = new HashMap<>()`; `@UpdateTimestamp`.

### B12
**`public void User(Long, String)` isn't a constructor** · Phase 2 · Code
- **Root cause:** it has a return type, and the class is named `Users`. Java allows methods named like a class, so it compiled silently.
- **Fix:** `public Users(Long githubId, String login)`.

### B13
**`No property 'existsUser' found for type 'Users'`** · Phase 2 · Code
- **Root cause:** Spring Data derived queries must follow `existsBy<Property>`; anything else is parsed as a property name.
- **Fix:** removed it; `existsById` and `findById` are inherited from `JpaRepository`.
- **Lesson:** repository methods are validated at startup, after the entity schema check.

### B14
**Never-synced users would be skipped by the sync query** · Phase 2 · Logic (found in review)
- **Root cause:** `findByNeedsReauthFalseAndLastSyncedAtBefore`: `NULL < cutoff` isn't true in SQL. Adding `Or...IsNull` binds wrongly, because `OR` has lower precedence than `AND`.
- **Fix:** `@Query`: `needsReauth = false and (lastSyncedAt is null or lastSyncedAt < :cutoff)`.
- **Lesson:** derived method names can't express brackets.

### B15
**IntelliJ: `Cannot resolve symbol 'log'` / `getGithubId`, while `./mvnw compile` passed** · Phase 2 · Tooling
- **Root cause:** Arch's `intellij-idea-community-edition` (IC-262.10315.125) doesn't bundle the Lombok plugin. Annotation processing settings were already correct.
- **Fix:** installed JetBrains' Lombok plugin 262.8665.176 (compatible with `262.*`) into the IDE's user plugin folder.
- **Lesson:** IDE errors + green Maven build = IDE setup problem. An IntelliJ update may require reinstalling the plugin from the Marketplace.

### B16
**`createdAt=null` right after `save()`** · Phase 2 · Code
- **Root cause:** the DB sets it (`DEFAULT now()`), but Hibernate doesn't read back columns it didn't write (`insertable = false`).
- **Fix:** `@Generated(event = EventType.INSERT)`.

### B17
**`LazyInitializationException: Could not initialize proxy [Users#...] - no session`** · Phase 2 · Code
- **Root cause:** `findById` runs in its own transaction; `Insights.user` is a LAZY proxy, and the session was closed when `getLogin()` tried to load it.
- **Fix:** `@EntityGraph(attributePaths = "user")` on `InsightsRepository.findById`. Also set `spring.jpa.open-in-view: false`, so web requests behave the same instead of hiding this.
- **Lesson:** load what you need inside the transaction (fetch graph / fetch join / service `@Transactional`).

### B18
**Re-running `save(new ...)` on existing rows** · Phase 2 · Code
- **Symptoms:** `Users`: no error, but `merge` copied the new object's default preferences over the stored row (a silent overwrite). `Insights`: `duplicate key value violates unique constraint "insights_pkey"`.
- **Root cause:** Spring Data's `save()` decides "new" by the id. `Users` had its id set → `merge`; `Insights` had a null id until `@MapsId` filled it → `persist` → INSERT.
- **Fix:** find-or-create (`findById(id).orElseGet(...)`), then modify and save.
- **Lesson:** for existing rows, always load and then update. Never `save(new Entity(existingId))`.

### B19
**`RestClient.Builder` auto-config only present through the Spring AI starter** · Phase 3 · Dependency (latent, no failure yet)
- **Found by:** `./mvnw dependency:tree` while building the GitHub client. `spring-boot-restclient` (which auto-configures `RestClient.Builder` in Boot 4) came in only via `spring-ai-starter-model-openai → spring-boot-starter-restclient`.
- **Risk:** removing or changing the AI starter would make startup fail with "no bean of type `RestClient.Builder`" in code unrelated to AI.
- **Fix:** declared `spring-boot-starter-restclient` directly in `pom.xml`.
- **Lesson:** Boot 4 splits auto-config into modules. Declare a starter for every feature you use directly; don't rely on transitive ones.

### B20
**v1: `calculate_user_insights` crashes on any empty repository** · Phase 4 · Logic (bug in v1, found while porting)
- **Symptom:** running v1's real Python code on the Phase 3 fixture: `AttributeError: 'NoneType' object has no attribute 'get'` at `repo.get("defaultBranchRef", {}).get("target", {})`.
- **Root cause:** GitHub returns `"defaultBranchRef": null` for a repo with no commits. Python's `dict.get(key, default)` only uses the default when the key is *missing*, not when its value is `null`.
- **Impact in v1 production:** `routes.py` swallows the exception in its broad `try/except`, so any user with an empty repo among their top 5 by stars logs in fine but **never gets insights saved**, silently.
- **Fix (v2):** `InsightsEngine` treats every nested GitHub object as nullable; covered by `InsightsEngineTest.emptyRepositoryDoesNotCrash_B20`. v1 itself is unchanged.
- **Lesson:** `x.get(k, {})` is not null-safe in Python, and a blanket `except` turns crashes into silent data loss.

### B21
**Phase 4 branch created from a stale `main`** · Phase 4 · Process
- **Symptom:** the Phase 3 test fixture was missing on the new branch.
- **Root cause:** local `main` was 4 commits behind `origin/main` (the PR #2 merge wasn't pulled), and the branch was created from the Phase 3 branch tip instead of the merge. The checkout had also changed between commands (likely IDE git activity during the merge).
- **Fix:** fast-forwarded `main` (`git merge --ff-only origin/main`) and recreated the branch from `origin/main` before any commits.
- **Lesson:** before branching, verify `git rev-parse --abbrev-ref HEAD` and `git status -sb` show `main` level with `origin/main`.

### B22
**`reference to assertThat is ambiguous`** · Phase 4 · Test code
- **Root cause:** `JsonMapper.valueToTree(...)` returns a generic `<T extends JsonNode> T`. Passed straight to AssertJ's overloaded `assertThat`, the compiler can't infer `T` (both `Predicate` and `IntPredicate` overloads match).
- **Fix:** assign the result to a `JsonNode` variable first.
