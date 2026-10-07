# 09 — Reactive

## M32. Reactive Programming and Project Reactor

### T1. Why reactive — `MK`
- The limits of thread-per-request; non-blocking I/O; backpressure; Netty event loops.
- `[3.x]` Reactive vs virtual threads.

### T2. Async three ways — `MK` `[P→S→B]`
- A `CompletableFuture` chain → a `Flux` / `Mono` pipeline → a WebFlux endpoint.

### T3. The Reactive Streams spec — `MK`
- `Publisher`, `Subscriber`, `Subscription`, `Processor`; `request(n)`; `java.util.concurrent.Flow`.

### T4. Mono and Flux — `MK`
- Creating them; cold vs hot publishers.
- Assembly vs subscription: nothing happens until you subscribe.

### T5. Operators — `MK`
- `map`, `flatMap`, `concatMap`, `flatMapSequential`, `filter`, `zip`, `merge`.
- `switchIfEmpty`, `defaultIfEmpty`, `collectList`.
- `GTH` `buffer` / `window`.

### T6. Error handling — `MK`
- `onErrorReturn`, `onErrorResume`, `onErrorMap`.
- `retryWhen(Retry.backoff(...))`, `timeout`.

### T7. Schedulers and threading — `MK`
- `publishOn` vs `subscribeOn`; `boundedElastic` for blocking calls; `parallel`.
- `GTH` BlockHound for detecting blocking calls.

### T8. Backpressure — `MK`
- Buffer, drop and latest strategies; `limitRate`.

### T9. Context — `MK`
- Reactor `Context` vs `ThreadLocal`; MDC propagation.
- `[3.x]` The context-propagation library.

### T10. Testing — `MK`
- `StepVerifier`, virtual time.
- `GTH` `TestPublisher`.

### T11. Debugging — `GTH`
- `checkpoint()`, the debug agent.

### Traps
- Blocking on the event loop.
- Nested `subscribe()` calls.
- Subscribing twice re-executes the whole pipeline.

---

## M33. Spring WebFlux

### T1. Architecture — `MK`
- `DispatcherHandler` with its handler mappings, adapters and result handlers; Netty by default; `WebFilter`.
- Comparison table: WebFlux vs MVC.

### T2. Annotated controllers — `MK`
- `/clients` endpoints returning `Mono` / `Flux`.

### T3. Functional endpoints — `MK`
- `RouterFunction`, `HandlerFunction`, `ServerRequest` / `ServerResponse`.

### T4. Error handling — `MK`
- `@ExceptionHandler`, `WebExceptionHandler`, `ErrorWebExceptionHandler`.
- `[3.x]` `ProblemDetail`.

### T5. WebClient in depth — `MK`
- `ExchangeFilterFunction`; Reactor Netty connection-pool and timeout tuning; retries.

### T6. Streaming — `GTH`
- Server-Sent Events, NDJSON, reactive WebSockets.

### T7. Reactive data and transactions — `GTH`
- A pointer to R2DBC; `TransactionalOperator` and `@Transactional` on reactive methods.

### T8. Security in WebFlux — `GTH`
- `SecurityWebFilterChain` and `ReactiveSecurityContextHolder`.

### T9. Testing — `MK`
- `WebTestClient` and `@WebFluxTest`.

### T10. Choosing a stack — `MK`
- WebFlux vs MVC vs MVC with virtual threads.

### Traps
- Blocking JDBC calls inside WebFlux.
- With both MVC and WebFlux on the classpath, Boot picks MVC.
