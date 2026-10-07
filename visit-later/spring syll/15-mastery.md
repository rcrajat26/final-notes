# 15 — Mastery

## M50. Migrating 2.7 → 3.x (and What 4.x Changes)

### T1. Prerequisites — `MK`
- Java 17; upgrade to the latest 2.7; fix deprecations; `spring-boot-properties-migrator`.

### T2. The Jakarta namespace — `MK`
- `javax` → `jakarta`; checking third-party library compatibility.
- `GTH` OpenRewrite recipes.

### T3. Framework 6 changes — `MK`
- Trailing-slash matching removed; `HttpStatusCode`; `ProblemDetail`; HTTP interface clients; the Observation API.

### T4. Boot 3 changes — `MK`
- `AutoConfiguration.imports` only; `@ConstructorBinding` moved; actuator property renames; Sleuth → Micrometer Tracing; Hibernate 6.

### T5. Security 6 changes — `MK`
- `requestMatchers`, `authorizeHttpRequests`, `@EnableMethodSecurity`.

### T6. Ecosystem upgrades — `MK`
- Spring Cloud 2022.0+, springdoc 2.x, `resilience4j-spring-boot3`, Spring Kafka 3.

### T7. What 4.x changes — `MK` `[4.x]`
- Framework 7 and Jackson 3.
- Modular auto-configuration and starters; JSpecify null-safety.
- API versioning, HTTP service groups, built-in resilience, Security 7.

### T8. The stale-answer sweep — `MK`
- Advice that's now wrong: `spring.factories` for auto-config, JDK proxies by default, circular references allowed, `javax`, `@MockBean`, `WebSecurityConfigurerAdapter`, Sleuth, `RestTemplate` as the recommended client.

---

## M51. Anti-patterns and When Not to Use Spring

### T1. The anti-pattern catalogue — `MK`
- Field injection everywhere; `ApplicationContextAware` used as a service locator; a god `@Configuration` class.
- Stateful singletons; `@Transactional` on repositories or around HTTP calls.
- Swallowing exceptions inside a transaction; secrets in profile files.
- `allow-circular-references=true`; `@EnableWebMvc` in a Boot app; scanning the `com` package.

### T2. Enforcing at build time — `GTH`
- ArchUnit rules and custom post-processor checks.

### T3. When not to use Spring — `GTH`
- CLIs, Lambda cold-start budgets, libraries.

### T4. The build-time DI contrast — `GTH`
- Quarkus and Micronaut.

---

## M52. Ecosystem Electives — `GTH` (pick any)

### T1. Spring Modulith
- Module boundaries; `@ApplicationModuleListener`; the event publication registry.

### T2. Spring Batch
- Jobs, steps, chunks, restartability.

### T3. Spring Integration
- Enterprise integration patterns.

### T4. Spring GraphQL

### T5. Spring Session

### T6. Deferred data track
- Spring Data JPA, Flyway, `JdbcTemplate` / `JdbcClient`, R2DBC.

### T7. Spring AI

---

## M53. Capstone — `MK`
Build `client-service` and `account-service` end to end.

### T1. Stage 1 — plain Java
- Hand-wired, with a hand-written HTTP server and hand-written transactions.

### T2. Stage 2 — Spring
- A Spring container with MVC, AOP auditing and declarative transactions.

### T3. Stage 3 — Boot 2.7
- Auto-config and configuration properties.
- REST with validation and error handling; JWT security.
- Kafka `AccountStatusChanged` events; inter-service HTTP calls with resilience.

### T4. Stage 4 — Production readiness
- Observability (metrics, tracing, correlated logs) and health probes.
- Graceful shutdown; a Testcontainers test suite.

### T5. Stage 5 — Migrate to Boot 3.x
- `RestClient`, `@ServiceConnection`, Micrometer Tracing, `ProblemDetail`, virtual threads.
- A closing note on the 4.x deltas.
