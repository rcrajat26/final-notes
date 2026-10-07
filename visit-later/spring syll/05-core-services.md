# 05 — Core Container Services

## M14. Application Events

### T1. Events three ways — `MK` `[P→S→B]`
- A hand-rolled observer pattern → `ApplicationEventPublisher` → Boot's lifecycle events.

### T2. Publishing and listening — `MK`
- `publishEvent(Object)`: any object works as an event since 4.2.
- `ApplicationListener` vs `@EventListener`; SpEL `condition`; listeners that return new events.
- `@Order`; generic events and `ResolvableTypeProvider`.

### T3. Synchronous by default — `MK`
- Listeners run on the publisher's thread, inside the publisher's transaction.
- An exception in a listener propagates back and can roll the publisher back.

### T4. Transactional events — `MK`
- `@TransactionalEventListener` phases; `fallbackExecution` (without it, the listener is skipped when there is no transaction).
- A database write inside an `AFTER_COMMIT` listener needs `REQUIRES_NEW`.
- Example: `AccountSuspendedEvent` → send the notification after commit.

### T5. Async events — `MK`
- `@Async` listeners.
- The multicaster's `taskExecutor` (makes all events async) and `errorHandler`.

### T6. Context and Boot events — `MK`
- `ContextRefreshedEvent` and `ContextClosedEvent`.
- Boot's sequence, from `ApplicationStartingEvent` to `ApplicationReadyEvent` / `ApplicationFailedEvent`; availability events.
- Events fired before the context exists can only be heard by listeners registered via `spring.factories` or `addListeners`.

### T7. Events vs messaging — `MK`
- Events are in-process and lost on a crash; listeners must be idempotent.
- The durable route: outbox → Kafka (see M36).
- `GTH` Spring Modulith (see M52).

### T8. Build it — `GTH`
- A mini event multicaster.

### Traps
- Listeners on a `@Lazy` bean are never registered.
- `ContextRefreshedEvent` can fire twice when there's a child context.

---

## M15. Environment, Profiles and Resources

### T1. Configuration three ways — `MK` `[P→S→B]`
- `System.getProperty`, `getenv` and `Properties` files → `Environment` → Boot config data.

### T2. Environment API — `MK`
- `PropertyResolver` methods.
- `PropertySource` and `MutablePropertySources` ordering: first in the list wins.
- The standard sources: system properties beat environment variables.

### T3. @PropertySource and placeholders — `MK`
- Placeholder syntax: defaults, nesting, and the unresolved-placeholder error.
- `@PropertySource` cannot read YAML.
- `PropertySourcesPlaceholderConfigurer`.

### T4. Profiles — `MK`
- `@Profile` expressions; active / default / include profiles; profile groups (2.4+).
- Use profiles to select behaviour, not to hold secrets.

### T5. Resources — `GTH`
- The `Resource` abstraction and its implementations.
- `classpath:`, `classpath*:` and `file:` prefixes; `ResourceLoader` and pattern resolution.
- Injecting a `Resource` via `@Value`.
- `MK` trap: `getFile()` fails inside a fat jar — use `getInputStream()`.

### Traps
- Putting secrets in profile-specific files committed to git.

---

## M16. SpEL and Type Conversion

### T1. @Value — `MK`
- `${...}` (property placeholder) vs `#{...}` (SpEL), and the order they're resolved in.
- Splitting a property into a list.
- Field injection timing: a `@Value` field isn't set yet inside the constructor.

### T2. The SpEL language — `GTH`
- Operators, collections, `T()` type references, `@bean` references, safe navigation, Elvis, selection and projection.
- The parser API.

### T3. Where SpEL appears — `MK`
- `@Value`, `@ConditionalOnExpression`, `@EventListener(condition)`, `@Cacheable`, `@PreAuthorize`, `@Scheduled`.

### T4. SpEL security — `MK`
- `StandardEvaluationContext` vs `SimpleEvaluationContext`.
- Never evaluate user input: it's a remote-code-execution vector.

### T5. Type conversion — `GTH`
- The legacy `PropertyEditor`.
- `Converter`, `ConverterFactory`, `GenericConverter`; `ConversionService` (the bean must be named `conversionService`).
- `Formatter`, `@DateTimeFormat`, `@NumberFormat`.

### T6. Conversion in use — `MK`
- A `String` → `AccountType` path-variable converter.
- Converters for configuration binding (`@ConfigurationPropertiesBinding`).

### Traps
- SpEL in hot cache keys is slow.
- `@Value` on a `static` field stays `null`.

---

## M17. Validation and Internationalization

### T1. Validation three ways — `MK` `[P→S→B]`
- Manual `if` checks → the Bean Validation API → Spring integration → Boot's `spring-boot-starter-validation` (a separate starter since 2.3).

### T2. Bean Validation essentials — `MK`
- Constraints on `Client` (email, phone pattern).
- `@Valid` cascading into the accounts list.
- A custom `@ValidPhone` constraint; validation groups.

### T3. Spring integration — `MK`
- `LocalValidatorFactoryBean`.
- Method validation with `@Validated` (which makes the bean a proxy).
- `MethodArgumentNotValidException` vs `ConstraintViolationException`.
- `[3.x]` `javax.validation` → `jakarta.validation`; built-in controller method validation in Framework 6.1.

### T4. Spring's Validator interface — `GTH`
- `Validator`, `Errors`, `DataBinder`.

### T5. Internationalization — `GTH`
- `MessageSource` implementations and `spring.messages.*`.
- `LocaleResolver` (Accept-Header, session, cookie) and `LocaleChangeInterceptor`.
- Localized validation messages: `ValidationMessages.properties` vs `messages.properties`.
- Localized error responses.

### Traps
- `@Validated` on a service brings every proxy limitation with it.
- The two validation exceptions need two different handlers.

---

## M18. Task Execution and Scheduling

### T1. Async work three ways — `MK` `[P→S→B]`
- `ExecutorService` / `ScheduledExecutorService` by hand → `@Async` / `@Scheduled` → Boot's task execution and scheduling auto-configuration.

### T2. @Async — `MK`
- `@EnableAsync`.
- Return types: `void`, `Future`, `CompletableFuture`; `[3.x]` `ListenableFuture` is deprecated.
- `AsyncUncaughtExceptionHandler`; choosing an executor by qualifier; proxy limits.

### T3. Boot's executors — `MK`
- The `applicationTaskExecutor` bean; `[3.x]` the `taskExecutor` alias was removed in 3.5.
- Defaults: core pool size 8, unbounded queue — so the pool never grows past core size.
- A `TaskDecorator` to propagate MDC and security context.

### T4. @Scheduled — `MK`
- Spring cron has 6 fields (it includes seconds).
- `fixedDelay` vs `fixedRate`; `initialDelay`; time `zone`.
- The default scheduler has exactly 1 thread.
- What happens when a scheduled method throws.

### T5. Scheduling across replicas — `MK`
- Every replica runs the job. Fixes: ShedLock, or an external scheduler.
- `GTH` Quartz.

### T6. Virtual threads — `GTH` `[3.x]`
- `spring.threads.virtual.enabled` (3.2+); `spring.main.keep-alive`.

### Traps
- Exceptions from a `void` `@Async` method are swallowed.
- A slow job blocks every other scheduled job on the single scheduler thread.

---

## M19. Cache Abstraction

### T1. Caching three ways — `MK` `[P→S→B]`
- A `ConcurrentHashMap` memo by hand → `@Cacheable` → Boot's cache auto-configuration.

### T2. Annotations — `MK`
- `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching`, `@CacheConfig`.
- SpEL keys; `condition` (checked before the call) vs `unless` (checked after); `sync`.

### T3. Keys — `MK`
- `SimpleKeyGenerator`'s rules, and the collisions they cause when two methods share a cache.
- A custom `KeyGenerator`.

### T4. Providers — `GTH`
- `spring.cache.type`; Caffeine and Redis at a high level; caching `null` values.

### T5. Proxies and ordering — `MK`
- Self-invocation skips the cache.
- Ordering the cache advisor against the transaction advisor.

### T6. JSR-107 annotations — `GTH`

### Traps
- Caching mutable objects.
- Two methods sharing a cache name with the same single argument.
