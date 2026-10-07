# 08 — HTTP Clients and Resilience

## M28. HTTP Clients

### T1. Calling another service three ways — `MK` `[P→S→B]`
- `client-service` → `account-service` using `java.net.http.HttpClient` → `RestTemplate` → Boot's `RestTemplateBuilder`.

### T2. RestTemplate — `MK`
- `exchange` and `getForObject`; `ResponseErrorHandler`; interceptors.
- Request factories: JDK, Apache HttpClient, OkHttp; timeouts.
- It is in maintenance mode.

### T3. WebClient for blocking apps — `MK`
- The builder; `retrieve` vs `exchangeToMono`; timeouts and error handling.
- Calling `.block()` inside an MVC app; the auto-configured `WebClient.Builder`.

### T4. RestClient — `MK` `[3.x]`
- Boot 3.2+: a fluent synchronous API and `RestClient.Builder`.
- Migrating from `RestTemplate`.

### T5. HTTP interface clients — `MK` `[3.x]`
- `@HttpExchange` / `@GetExchange` with `HttpServiceProxyFactory`.
- `[4.x]` `@ImportHttpServices` group configuration.

### T6. OpenFeign — `MK`
- `@FeignClient`; configuration, error decoders, interceptors; its current status.

### T7. Cross-cutting concerns — `MK`
- Connection pooling; connect, read and response timeouts.
- Propagating auth headers.
- Building clients from Boot's builders so observability is applied (see M40).

### T8. Comparison — `MK`
- Table: sync vs async, maintenance status, when to use each client.

### T9. Testing clients — `MK`
- `MockRestServiceServer`, `@RestClientTest`, WireMock, MockWebServer.

### Traps
- No timeouts by default.
- Creating a `RestTemplate` per request.
- Clients created with `new` bypass instrumentation.

---

## M29. Spring Retry

### T1. Retry three ways — `MK` `[P→S→B]`
- A hand-written retry loop → `RetryTemplate` → `@Retryable`.

### T2. Annotations — `MK`
- `@EnableRetry`, `@Retryable` (exception types, `maxAttempts`), `@Backoff` (delay, multiplier, maxDelay, random), `@Recover`.

### T3. RetryTemplate — `MK`
- Retry policies (simple, timeout) and backoff policies (fixed, exponential random).
- `RetryListener`; stateful vs stateless retry.

### T4. What to retry — `MK`
- Only idempotent operations; transient vs permanent errors; retry budgets.
- Ordering retry against transactions.

### T5. Status — `MK`
- Spring Retry is in maintenance mode.
- `[4.x]` Framework 7 has built-in `@Retryable`, `@ConcurrencyLimit` and `RetryTemplate`.

### Traps
- Self-invocation skips retry.
- Retrying a non-idempotent POST.
- A `@Recover` method whose signature doesn't match is never called.

---

## M30. Resilience4j and Spring Cloud CircuitBreaker

### T1. Patterns — `MK` (condensed)
- Circuit breaker states, bulkhead, rate limiter, time limiter, retry — and the order to compose them in.

### T2. Resilience4j in Boot — `MK`
- `resilience4j-spring-boot2` vs `[3.x]` `resilience4j-spring-boot3`.
- Annotations with `fallbackMethod`; aspect ordering; YAML instance configuration.

### T3. Spring Cloud CircuitBreaker — `MK`
- The `CircuitBreakerFactory` abstraction, the Resilience4j implementation, customizers.

### T4. Observability — `MK`
- Actuator endpoints, Micrometer metrics, health indicators.

### T5. Choosing — `MK`
- Decision table: Resilience4j vs Spring Retry vs Framework 7's built-ins.

### Traps
- A fallback that hides real outages.
- A time limiter on synchronous code doesn't actually interrupt the call.

---

## M31. Spring Cloud Service Patterns

### T1. Release trains — `MK`
- 2021.0.x pairs with Boot 2.7; `[3.x]` 2022.0+ with Boot 3; the compatibility matrix.

### T2. Centralized configuration — `MK`
- Config Server and Config Client.
- `spring.config.import=configserver:` vs the legacy `bootstrap.yml` (bootstrap context).
- `@RefreshScope` and `/actuator/refresh`.
- `GTH` Spring Cloud Bus.

### T3. Discovery and load balancing — `GTH`
- Eureka / Consul; Spring Cloud LoadBalancer; `@LoadBalanced` clients.
- Kubernetes-native alternatives.

### T4. API gateway — `GTH`
- Spring Cloud Gateway (built on WebFlux): routes, predicates, filters, rate limiting.

### T5. Spring Cloud AWS — `GTH`
- Parameter Store and Secrets Manager config import; SQS (see M37).

### T6. Distributed patterns — `GTH`
- Pointers to saga and outbox; Spring Cloud Contract (see M45).

### T7. The retired stack — `MK`
- Hystrix, Ribbon and Zuul are gone.
- `[3.x]` Sleuth is replaced by Micrometer Tracing.

### Traps
- Mismatched Boot and Spring Cloud versions.
- `@RefreshScope` beans losing state on refresh.
