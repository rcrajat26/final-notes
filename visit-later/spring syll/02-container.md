# 02 — The Container

## M05. Bean Scopes

### T1. Singleton — `MK`
- One instance per container, not per JVM.
- Keep singletons stateless. Demo: a data race caused by a mutable field.

### T2. Prototype — `MK`
- A new instance per lookup, with no destroy callbacks.
- Injecting a prototype into a singleton captures one instance forever. Fixes, ranked: `ObjectProvider` → `@Lookup` → scoped proxy.

### T3. Web scopes — `MK` `[P→S→B]`
- `request`, `session` and `application` scopes; the `@RequestScope` family of shortcuts.
- How request-scoped state is bound: a `ThreadLocal` in `RequestContextHolder`, plus `RequestContextFilter`.
- Plain-Java side: servlet request attributes vs a request-scoped bean.

### T4. Scoped proxies — `MK`
- `ScopedProxyMode`, and why `TARGET_CLASS` is the safe choice.
- The proxy resolves the real target on every call; the real definition is renamed `scopedTarget.<name>`.

### T5. Custom scopes — `GTH`
- The `Scope` SPI and `registerScope`.
- Worked example: a tenant scope.

### T6. Scopes and threads — `MK`
- A request-scoped bean used from `@Async` or `@Scheduled` throws "No thread-bound request found".
- `[3.x]` How virtual threads change which threads carry request state.

### Traps
- Prototypes holding resources leak, because nobody calls `@PreDestroy`.
- Mutable state on a singleton is shared across requests.

---

## M06. Bean Lifecycle

### T1. The ordered lifecycle — `MK`
- Creation, in order:
  1. Instantiate (constructor and constructor injection).
  2. Populate (field and setter injection).
  3. `Aware` callbacks.
  4. Post-processors' before-initialization callback (`postProcessBeforeInitialization`).
  5. `@PostConstruct`.
  6. `afterPropertiesSet()`.
  7. The custom init method.
  8. Post-processors' after-initialization callback (`postProcessAfterInitialization`) — **this is where AOP proxies are created**.
  9. `SmartInitializingSingleton`, once all singletons exist.
- Then the bean is in use.
- Destruction runs in reverse dependency order.

### T2. Aware interfaces — `MK`
- `BeanNameAware`, `BeanFactoryAware`, `ApplicationContextAware`, `EnvironmentAware`, `ResourceLoaderAware`, `ApplicationEventPublisherAware`.
- Prefer plain injection over `Aware` interfaces.
- `GTH` `ImportAware`.

### T3. Init and destroy — `MK`
- The three mechanisms and their fixed order; `@Bean(initMethod, destroyMethod)`.
- `@Bean` infers a destroy method: a public `close()` or `shutdown()` is called automatically.
- `[3.x]` `javax.annotation` → `jakarta.annotation`.

### T4. Lifecycle and SmartLifecycle — `MK`
- `start` / `stop` / `isRunning`, and phases: the lowest phase starts first and stops last.
- `DefaultLifecycleProcessor` waits 30 seconds per shutdown phase by default (`spring.lifecycle.timeout-per-shutdown-phase`).

### T5. Startup hooks compared — `MK`
- `@PostConstruct` vs `SmartInitializingSingleton` vs `ApplicationRunner` / `CommandLineRunner` vs an `ApplicationReadyEvent` listener.
- For each: what exists yet, and whether proxies and transactions work.

### T6. Shutdown — `MK`
- `close()` → `ContextClosedEvent` → `Lifecycle` beans stop → beans destroyed in reverse dependency order.
- The JVM shutdown hook — which never runs on SIGKILL.

### T7. Lazy initialization — `MK`
- `@Lazy`, and the global `spring.main.lazy-initialization`.
- The trade: faster startup, but the first request pays and configuration errors surface late.

### T8. Background bean initialization — `GTH` `[3.x]`
- Framework 6.2's `@Bean(bootstrap = BACKGROUND)`.

### Traps
- Calling a `@Transactional` method from `@PostConstruct` gets no transaction: the proxy doesn't exist yet.
- `@PreDestroy` is never called on prototypes.
- Side effects in a proxied bean's constructor may run twice or not at all.

---

## M07. Container Extension Points

### T1. The two-phase model — `MK`
- **Definition phase:** post-processors mutate bean definitions; no beans exist yet.
- **Instance phase:** post-processors see and wrap actual objects.
- Every extension question reduces to "which phase?".

### T2. Factory post-processors — `MK`
- `BeanFactoryPostProcessor` and `BeanDefinitionRegistryPostProcessor` (the latter can add new definitions).
- `PropertySourcesPlaceholderConfigurer` as the canonical example; why it must be declared as a `static @Bean`.

### T3. The BeanPostProcessor family — `MK`
- `BeanPostProcessor`: before and after initialization.
- `InstantiationAwareBeanPostProcessor`: can short-circuit creation entirely.
- `SmartInstantiationAwareBeanPostProcessor`: picks constructors and supplies early references (needed for AOP plus cycles).
- `GTH` `MergedBeanDefinitionPostProcessor` and `DestructionAwareBeanPostProcessor`.

### T4. Built-in processors — `MK`
- Container and injection:
  - `ConfigurationClassPostProcessor`.
  - `AutowiredAnnotationBeanPostProcessor`, `CommonAnnotationBeanPostProcessor`.
- Proxy creation: the auto-proxy creators.
- Feature annotations:
  - The async and scheduling processors.
  - `ConfigurationPropertiesBindingPostProcessor`.
  - `EventListenerMethodProcessor`.
  - `PersistenceExceptionTranslationPostProcessor`.

### T5. Ordering — `MK`
- `PriorityOrdered` runs first, then `Ordered`, then the rest.
- Any bean a post-processor depends on is created early and can't be proxied. Log signature: "not eligible for getting processed by all BeanPostProcessors".

### T6. Boot-level hooks — `MK`
- `EnvironmentPostProcessor`: the right place to add a secrets property source.
- `GTH` `ApplicationContextInitializer`, `SpringApplicationRunListener`, `FailureAnalyzer`.
- Registered via `META-INF/spring.factories`, which is still used for these (only auto-configuration moved out).

### T7. Choosing an extension point — `MK`
- Decision table: mutate a definition / wrap an instance / add definitions / add property sources / run after startup.

### T8. Build it — `GTH`
- A post-processor that fails startup when `@Transactional` appears on a `private` method.
- A per-bean initialization timer.

### Traps
- `@Autowired` fields inside a post-processor are unreliable; wire it through `@Bean` method parameters instead.

---

## M08. @Configuration and @Bean

### T1. @Bean semantics — `MK`
- The return type is the bean type, the method name is the bean name, and parameters are injected.
- Its attributes.

### T2. Full vs lite mode — `MK`
- In a `@Configuration` class, calls between `@Bean` methods return the container's singleton (the class is CGLIB-enhanced).
- `proxyBeanMethods=false` (since 5.2) turns this off; Boot's own auto-configurations run this way.
- Prefer taking dependencies as method parameters.

### T3. Constraints — `MK`
- The class must not be `final`, and `@Bean` methods must not be `private` or `final`.
- Use `static @Bean` methods for post-processors.

### T4. Conditional configuration — `MK`
- `@Conditional` and the `Condition` interface.
- `@Profile` is just a condition.
- `GTH` `ConfigurationPhase`, and overloaded `@Bean` methods combined with `@Profile`.

### T5. Factory class vs @Configuration vs auto-config — `MK` `[P→S→B]`

### Traps
- In lite mode, an inter-method call creates a second instance.
- A `final` `@Configuration` class fails at startup.

---

## M09. Container Internals

### T1. refresh(), step by step — `MK`
- All twelve steps, in order.
- Boot's hooks: the web server is created in `onRefresh` but only started in `finishRefresh`.
- On failure, beans already created are destroyed.

### T2. The bean creation path — `MK`
- The chain: `getBean` → `doGetBean` → `createBean` → `resolveBeforeInstantiation` → `doCreateBean`.
- Inside `doCreateBean`:
  1. Instantiate.
  2. Merged-definition post-processing.
  3. Early reference exposure.
  4. Populate.
  5. Initialize.
  6. Register destruction callbacks.
- Reading a nested `BeanCreationException` cause chain.

### T3. The three-level singleton cache — `MK`
- The three maps: `singletonObjects`, `earlySingletonObjects`, `singletonFactories`; the `getSingleton` walk.
- Proof, walked step by step: why a field cycle (A→B→A) resolves and a constructor cycle cannot.
- Why three levels, not two: the AOP early reference produced via `getEarlyBeanReference`.
- The "injected in its raw version… but has eventually been wrapped" error, and `@Async` beans in a cycle.

### T4. Injection internals — `GTH`
- How `AutowiredAnnotationBeanPostProcessor` caches injection metadata and picks candidate constructors.

### T5. Configuration parsing — `GTH`
- `ConfigurationClassPostProcessor`'s parse order, and CGLIB enhancement of `@Configuration` classes.

### T6. Thread safety — `MK`
- The container safely publishes initialized singletons; it does nothing for your mutable fields.
- Concurrent `getBean` calls.
- `GTH` Reading a startup deadlock in a thread dump.

### T7. Diagnostics — `MK`
- `/actuator/beans`; the `org.springframework.beans.factory` DEBUG log category; finding where a bean was defined.

### T8. Build it: three-level cache — `MK`
- Extend the mini container with the three-level cache.
- Collapse it to two levels and watch what breaks.

### Traps
- An `@Async` bean in a cycle fails startup, while a `@Transactional` one doesn't.
- Assuming the container makes your beans thread-safe.
