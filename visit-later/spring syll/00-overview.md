# Spring & Spring Boot — Course Syllabus (v1)

## Baseline
- **Primary:** Spring Boot 2.7 (Spring Framework 5.3, `javax.*`, Spring Security 5.7, Spring Cloud 2021.0).
- **Flagged:** Spring Boot 3.x differences are marked `[3.x]` (Framework 6, `jakarta.*`, Java 17 baseline).
- **One-liners:** Spring Boot 4.x changes are marked `[4.x]` (Framework 7).
- **Java:** Java 17 for 2.7 code (2.7 runs up to Java 21). Java 21 only for 3.x-only features such as virtual threads.

## Teaching domain
```
Client  { name, email, phone, List<Account> accounts }
Account { status: ACTIVE | INACTIVE | SUSPENDED, type: IGCFD | IGSTK }
```
- **Services:** `client-service` and `account-service`. They run as two apps wherever inter-service behaviour matters (HTTP clients, resilience, Kafka, tracing, security).
- **Persistence:** in-memory repositories by default. Where a real database is needed, plain JDBC or `JdbcTemplate` is used *incidentally*, not taught.

## Conventions
| Tag | Meaning |
|---|---|
| `MK` | Must Know |
| `GTH` | Good to Have (subtopics inherit the topic's tag unless marked) |
| `[P→S→B]` | Same feature shown three ways: plain Java → Spring → Spring Boot. The plain-Java side is often a small "build it" implementation, to show exactly what is abstracted away |
| `[3.x]` / `[4.x]` | Version difference from the 2.7 baseline |
| **Traps** | Closing list per module: wrong belief → symptom → fix |

Other conventions:
- Teaching is concise by default, with depth on request.
- End-of-module interview questions are `GTH`, generated on request.
- Diagnostics (actuator endpoints, log categories, reading stack traces) are folded into each module rather than taught separately.

## Out of scope (soft rule — used where needed, never taught as a standalone topic)
- Spring Data JPA.
- Flyway and Liquibase.
- `JdbcTemplate` and `JdbcClient`.
- Docker images and buildpacks.

## File index
| File | Modules |
|---|---|
| 01-foundations.md | M01 History · M02 IoC & DI · M03 Bean definitions · M04 Dependency resolution |
| 02-container.md | M05 Scopes · M06 Lifecycle · M07 Extension points · M08 @Configuration · M09 Container internals |
| 03-aop.md | M10 Proxy model · M11 Spring AOP |
| 04-transactions.md | M12 Declarative transactions · M13 Transactions in practice & internals |
| 05-core-services.md | M14 Events · M15 Environment & resources · M16 SpEL & conversion · M17 Validation & i18n · M18 Task execution & scheduling · M19 Cache abstraction |
| 06-boot-core.md | M20 What Boot adds · M21 Auto-configuration · M22 Externalized configuration · M23 Logging |
| 07-web-mvc.md | M24 MVC foundations · M25 REST APIs · M26 Error handling & validation · M27 Web extension points & server |
| 08-http-clients-resilience.md | M28 HTTP clients · M29 Spring Retry · M30 Resilience4j & CircuitBreaker · M31 Spring Cloud patterns |
| 09-reactive.md | M32 Reactive & Reactor · M33 WebFlux |
| 10-security.md | M34 Spring Security in Boot · M35 OAuth2, JWT & method security |
| 11-messaging.md | M36 Spring Kafka · M37 Messaging beyond Kafka |
| 12-observability.md | M38 Actuator · M39 Micrometer metrics · M40 Tracing · M41 Health, probes & log correlation |
| 13-testing.md | M42 Test foundations & TestContext · M43 Slices · M44 Testcontainers · M45 Cross-cutting tests |
| 14-production.md | M46 Packaging · M47 Startup & shutdown · M48 Concurrency in Boot · M49 AOT & native |
| 15-mastery.md | M50 Migration 2.7→3.x→4.x · M51 Anti-patterns · M52 Electives · M53 Capstone |
