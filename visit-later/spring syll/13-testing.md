# 13 — Testing

## M42. Test Foundations and the TestContext Framework

### T1. Testing three ways — `MK` `[P→S→B]`
- Plain JUnit with manual wiring (the payoff of constructor injection) → Spring's TestContext → `@SpringBootTest`.

### T2. The test pyramid for Spring apps — `MK`
- Unit, slice, integration and contract tests.

### T3. @SpringBootTest — `MK`
- `webEnvironment` modes; `properties`, `classes`.
- `TestRestTemplate` / `WebTestClient`; `@LocalServerPort`.

### T4. The TestContext framework — `MK`
- `SpringExtension`, `@ContextConfiguration`, `@TestPropertySource`, `@DynamicPropertySource`, `@ActiveProfiles`.
- Test execution listeners.

### T5. Context caching — `MK`
- What goes into the cache key.
- Bean overrides fork a new context.
- The cache holds 32 contexts by default; DEBUG statistics logging.
- The cost of `@DirtiesContext`; forked JVMs disable the cache entirely.

### T6. Bean overriding — `MK`
- `@MockBean` / `@SpyBean` → `[3.x]` `@MockitoBean` / `@MockitoSpyBean` (Boot 3.4+).
- `@TestConfiguration`, `@Import`.

### T7. Mockito essentials for Spring — `MK` (brief)

### Traps
- A `@MockBean` sprinkled in every class means one new context per class.

---

## M43. Test Slices

### T1. How slices work — `MK`
- The bootstrapper, type-exclude filters and a curated auto-configuration list; the `@AutoConfigure*` annotations.

### T2. @WebMvcTest — `MK`
- `MockMvc`; what's included (controllers, advice, filters); mocking services.
- `[3.x]` `MockMvcTester` (3.4+).

### T3. @WebFluxTest — `MK`
- `WebTestClient`.

### T4. @JsonTest — `MK`
- `JacksonTester`.

### T5. @RestClientTest — `MK`
- `MockRestServiceServer`.

### T6. Data slices — `GTH`
- Pointers to `@DataJpaTest` and `@JdbcTest` (deferred topics).

### T7. Custom slices — `GTH`

### Traps
- Security filters active in `@WebMvcTest` cause unexpected 401s.

---

## M44. Testcontainers and Service Connections

### T1. Why Testcontainers — `MK`
- Real Postgres and Kafka vs H2 and embedded fakes.

### T2. The 2.7 style — `MK`
- `@Testcontainers` / `@Container`, static containers.
- Wiring with `@DynamicPropertySource`; the singleton-container pattern; container reuse.

### T3. @ServiceConnection — `MK` `[3.x]`
- Boot 3.1+: the `ConnectionDetails` abstraction, the supported containers, `@ImportTestcontainers`.

### T4. Dev-time containers — `GTH` `[3.x]`
- `SpringApplication.from(...).with(...)`; Docker Compose support.

### T5. Multi-dependency tests — `MK`
- A Kafka container plus WireMock standing in for `account-service`.

### T6. Speed — `MK`
- Combining the context cache with container lifecycle.

### Traps
- Per-class containers combined with a cached context point at a dead port.

---

## M45. Testing Cross-cutting Behaviour

### T1. Transactions — `MK`
- `@Transactional` tests roll back; `@Commit`; `TestTransaction`.
- `AFTER_COMMIT` behaviour is never exercised this way.

### T2. Events — `MK`
- `@RecordApplicationEvents` with `ApplicationEvents`.

### T3. Async and scheduling — `MK`
- Awaitility; disabling schedulers in tests.

### T4. Security — `MK`
- See M35 T7.

### T5. Auto-configuration and starters — `MK`
- `ApplicationContextRunner`, `FilteredClassLoader`, context assertions.

### T6. HTTP clients and contracts — `MK`
- WireMock, MockWebServer.
- `GTH` Spring Cloud Contract.

### T7. Architecture tests — `GTH`
- ArchUnit rules: no field injection, no `@Transactional` on `private` methods, no entity return types.

### T8. Context-fork guard — `GTH`
- A test that fails when the suite starts more than N contexts.
