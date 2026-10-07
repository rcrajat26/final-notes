# 12 — Observability

## M38. Actuator

### T1. Operations endpoints three ways — `MK` `[P→S→B]`
- A hand-rolled `/health` and JMX MBeans → Spring Boot Actuator.

### T2. Endpoints — `MK`
- The common endpoints:
  - `health`, `info`, `metrics`, `prometheus`, `loggers`.
  - `env`, `configprops`, `beans`, `conditions`, `mappings`.
  - `threaddump`, `heapdump`, `startup`, `scheduledtasks`, `caches`.
- Exposure: web vs JMX, `management.endpoints.web.exposure.include`.
- A separate management port and base path.

### T3. Securing actuator — `MK`
- A dedicated security chain for actuator; sensitive endpoints.
- `[3.x]` Values in `env` / `configprops` are masked by default (`show-values`).

### T4. Info — `GTH`
- `build-info`, `git.properties`, custom `InfoContributor`s.

### T5. Custom endpoints — `GTH`
- `@Endpoint`, `@ReadOperation` / `@WriteOperation`, `@WebEndpoint`.

### Traps
- Exposing `env` or `heapdump` publicly.

---

## M39. Metrics with Micrometer

### T1. The Micrometer model — `MK` (condensed)
- `MeterRegistry` and the meter types.
- Tags and cardinality; naming conventions.

### T2. Boot auto-instrumentation — `MK`
- `http.server.requests` and `http.client.requests`.
- JVM, process, Hikari, executor, Logback and Tomcat metrics.

### T3. Registries — `MK`
- Prometheus and composite registries; `GTH` CloudWatch.
- `[3.x]` Export properties move from `management.metrics.export.prometheus.*` → `management.prometheus.metrics.export.*`.

### T4. Custom metrics — `MK`
- Example: a counter of suspended accounts.
- `@Timed` (needs a `TimedAspect` bean), `MeterBinder`, `MeterFilter` (common tags, deny rules).
- Percentiles, histograms and SLOs.

### T5. The Observation API — `MK` `[3.x]`
- `ObservationRegistry`, `@Observed`.
- One instrumentation produces both metrics and traces.

### T6. Dashboards and alerting — `GTH`
- Grafana; RED and USE methods.

### Traps
- Cardinality explosion from a `clientId` tag.
- `@Timed` without `TimedAspect` does nothing.

---

## M40. Distributed Tracing

### T1. Concepts — `MK`
- Traces and spans; context propagation; W3C `traceparent` vs B3 headers.

### T2. Boot 2.7: Spring Cloud Sleuth — `MK`
- Auto-instrumentation with Brave, reporting to Zipkin; baggage.

### T3. Boot 3: Micrometer Tracing — `MK` `[3.x]`
- Brave or OpenTelemetry bridges; Zipkin and OTLP exporters.
- `management.tracing.sampling.probability`; baggage.

### T4. Instrumentation coverage — `MK`
- Covered: MVC, WebFlux, HTTP clients built from Boot's builders, Kafka, `@Async`, scheduled tasks.

### T5. Custom spans — `MK`
- The `Tracer` API; `@NewSpan` in Sleuth vs Observations in 3.x.

### T6. Backends — `GTH`
- Tempo, Jaeger, the OpenTelemetry Collector.

### Traps
- Clients created with `new` lose trace propagation.
- Sampling set to 1.0 in production.

---

## M41. Health, Probes and Log Correlation

### T1. Health indicators — `MK`
- The built-in indicators.
- Custom indicators, e.g. for `account-service` as a downstream dependency.
- Status aggregation, `show-details`, health groups.

### T2. Liveness and readiness — `MK`
- `ApplicationAvailability` and `AvailabilityChangeEvent`.
- `/health/liveness` and `/health/readiness`; enabling probes.
- ECS and Kubernetes health checks.
- What *not* to put in liveness.

### T3. Log correlation — `MK`
- `traceId` / `spanId` in the MDC.
- The Sleuth log pattern in 2.7; `[3.x]` `logging.pattern.correlation` (3.2+); trace ids in structured logs.

### T4. Wiring the three pillars — `MK`
- Metrics → Prometheus, traces → Tempo, logs → Loki.
- `GTH` Exemplars.

### T5. Startup observability — `GTH`
- `ApplicationStartup`, `BufferingApplicationStartup`, `/actuator/startup`.

### Traps
- A downstream dependency in the liveness check causes cascading restarts.
