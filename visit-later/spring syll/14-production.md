# 14 — Production Runtime

## M46. Packaging and the Executable Jar

### T1. Packaging three ways — `MK` `[P→S→B]`
- `java -cp` with a `lib/` folder → a shaded uber jar → Boot's nested jar.

### T2. Fat jar layout — `MK`
- `BOOT-INF/classes` and `BOOT-INF/lib`, the loader classes.
- `MANIFEST.MF`: `Main-Class` is `JarLauncher`, `Start-Class` is your class.
- Nested jars are stored uncompressed.

### T3. The loader — `GTH`
- `JarLauncher` and its class loader.
- `[3.x]` The loader was rewritten in 3.2.

### T4. Build plugins — `MK`
- `repackage`, `run`, `build-info`.
- A brief look at the layer index.
- `[3.x]` `jarmode=layertools` → `jarmode=tools`.

### T5. Resources inside jars — `MK`
- The `getFile()` trap.

### Traps
- Relying on the filesystem for files that live in the jar.

---

## M47. Startup, Shutdown and Runtime Lifecycle

### T1. The startup budget — `MK`
- Where startup time goes, and how to measure it.

### T2. Startup levers — `MK`
- Narrow scanning, excluding auto-configurations, the lazy-init trade-off.
- `[3.x]` AOT; `GTH` CRaC (3.2+).

### T3. Graceful shutdown — `MK`
- `server.shutdown=graceful` (2.3+) and `spring.lifecycle.timeout-per-shutdown-phase`.
- The sequence: SIGTERM → shutdown hook → `ContextClosedEvent` → `SmartLifecycle` phases → executors and Kafka containers drain.
- Aligning with ECS `stopTimeout` / Kubernetes grace periods.

### T4. Executor and listener shutdown — `MK`
- `spring.task.execution.shutdown.*`; stopping Kafka listeners cleanly.

### T5. Diagnosing startup failures — `MK`
- The `FailureAnalyzer` output box.
- The most common failures: port in use, no DataSource configured, missing bean, circular reference.

### Traps
- An orchestrator kill timeout shorter than the drain time.
- Startup work placed in `@PostConstruct` instead of a runner.

---

## M48. Concurrency in Boot Apps

### T1. The thread model map — `MK`
- Tomcat request threads, `applicationTaskExecutor`, the scheduler, Kafka consumer threads, Netty event loops, `boundedElastic`.

### T2. Sizing and configuration — `MK`
- Property defaults, bounded queues, rejection policies.

### T3. Context propagation — `MK`
- MDC, security context, tracing and request scope across threads; `TaskDecorator`.
- `[3.x]` The context-propagation library.

### T4. Virtual threads — `MK` `[3.x]`
- `spring.threads.virtual.enabled` (3.2+): web servers, executors, schedulers and messaging listeners switch over.
- Pinning on `synchronized` (JDK 21–23); `keep-alive`.

### Traps
- Unbounded queues hide overload.
- `ThreadLocal` caches multiplied across virtual threads.

---

## M49. AOT and Native Images — `GTH`

### T1. Concepts
- AOT vs native image; the closed-world assumption.

### T2. Boot 2.7
- The experimental `spring-native` project.

### T3. Boot 3.x `[3.x]`
- `process-aot` and the generated initializers.
- `RuntimeHints`, `@RegisterReflectionForBinding`, `@ImportRuntimeHints`; `native-maven-plugin`.

### T4. Limitations
- A fixed classpath; profiles and conditions baked in at build time.
- Reflection, proxies and resources must be declared; CGLIB proxies are generated ahead of time.

### T5. Testing native builds

### T6. When it's worth it
- Short-lived processes and Lambda cold starts.
