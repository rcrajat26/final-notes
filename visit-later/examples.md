# ExecutorService in Practice — 3 IG Codebases

How `ExecutorService` / `ScheduledExecutorService` is actually used in:

1. **mtfi-document-service** — `_non-clinet-tech/mtfi-document-service`
2. **mtfi-instrument-management** — `_non-clinet-tech/mtfi-instrument-management`
3. **payments-gateway** — `payments/payments-gateway`

…and a recreate-it-yourself guide (dependencies, properties, annotations, code, tests) for each pattern.

---

## 0. Quick map

| # | Codebase | File | Pattern | Executor type |
|-|-|-|-|-|
| A | mtfi-document-service | `integration/.../FlipService.java` | Run blocking startup work off the Spring init thread | `Executors.newSingleThreadExecutor()` |
| B | mtfi-document-service | `domain/.../service/XmlGenerationService.java` | Debounced "run once in 5 minutes" job | Raw `new Thread(...)` (imports `ScheduledExecutorService` but doesn't use it) |
| C | mtfi-document-service | `integration/.../gateway/AwsFileDownloadService.java` | Parallel zip (commented out) | `Executors.newFixedThreadPool(nCPU)` + `CompletableFuture.runAsync` |
| D | mtfi-instrument-management | `integration/.../configuration/TaskSchedulerConfiguration.java` | Custom pool backing all `@Scheduled` jobs + dynamic cron jobs | `Executors.newScheduledThreadPool(n)` wrapped in `ConcurrentTaskScheduler` |
| E | mtfi-instrument-management | `redemption/RedemptionProcessService.java`, `schduler/RegisterWarrantService.java` | Dynamic, data-driven cron scheduling via `TaskScheduler.schedule(..., CronTrigger)` | Uses bean from D |
| F | mtfi-instrument-management | `test/.../UnderlyingReferencePriceServiceTest.java` | Testing `wait()/notify()` with 2 threads | `Executors.newFixedThreadPool(2)` |
| G | payments-gateway | `integration/.../web/service/ExternalVerificationRequestsServiceImpl.java` | **Parallel fan-out of 3 Feign calls with per-call timeout** | `Executors.newFixedThreadPool(10, daemonFactory)` + `CompletableFuture.supplyAsync` |
| H | payments-gateway | `integration/.../observability/logging/IGClusterDetails.java` | Poll a file every 1s for light/dark state | `Executors.newSingleThreadScheduledExecutor(daemonFactory)` |

The best pattern to copy is **G**, followed by **D + E**. **A** is a good small one. **B** and **F** are useful mainly as examples of what *not* to do.

---

## 1. mtfi-document-service

### 1A. `FlipService`: blue/green Kafka flip on a background thread

**What it does**

The service runs as two deployments, blue and green. Only the **light** (live) colour should consume Kafka. ZooKeeper stores the live colour at `${zookeeper.root.path}`. On startup, `FlipService`:

1. Reads the live colour from ZooKeeper synchronously (`curatorFramework.getData().forPath(...)`), which is a blocking network call.
2. Starts or stops every `KafkaMessageListenerContainer` depending on whether this instance's colour matches.
3. Registers a `TreeCache` listener so later flips (`NODE_ADDED` / `NODE_UPDATED`) start or stop the containers again.

The blocking ZooKeeper call and the listener wiring run on a **single-thread executor**. That keeps them off Spring's bean-initialisation thread, so a slow or unreachable ZooKeeper can't stall application context startup.

**Code (actual, trimmed)**

```java
@Slf4j
@Service
@DependsOn({                                   // containers must exist before we start/stop them
   "warrantInstrumentMessageListenerContainer",
   "isinMessageListenerContainer",
   "warrantRedemptionListenerContainer"
})
public class FlipService {

   private final ExecutorService executor = Executors.newSingleThreadExecutor();
   // + CuratorFramework, TreeCache, List<KafkaMessageListenerContainer>, ClusterDetails ...

   FlipService(CuratorFramework curatorFramework,
               @Value("${zookeeper.root.path}") String configRootPath,
               TreeCache cache,
               List<KafkaMessageListenerContainer> listOfContainer,   // Spring injects ALL container beans
               ClusterDetails clusterDetails,
               ZookeeperDarkOrLightManager zookeeperDarkOrLightManager) { ... }

   @PostConstruct
   public void start() {
      this.executor.execute(() -> {
         try {
            this.startService(this.checkZooKeeperIsLightSync());          // blocking ZK read
            this.cache.getListenable().addListener((client, event) -> {   // react to future flips
               if (event.getType() == TreeCacheEvent.Type.NODE_ADDED
                || event.getType() == TreeCacheEvent.Type.NODE_UPDATED) {
                  this.startService(this.isLight(event.getData().getData()));
               }
            });
         } catch (Exception e) {
            log.error("Failed to start FlipService TreeCache", e);
         }
      });
   }

   @PreDestroy
   public void stop() {
      this.executor.shutdown();
   }
}
```

**Supporting config**

- `KafkaConfiguration` creates the containers with `container.setAutoStartup(false)` so nothing consumes until `FlipService` decides.
- `LeadershipConfiguration` defines the `TreeCache` bean: `TreeCache.newBuilder(curatorClient, configRootPath).build()`. It also defines `ClusterDetails` and `ZookeeperDarkOrLightManager`, which come from the IG lib `com.iggroup.wt.zookeeper.leader.decider`.
- Properties: `zookeeper.root.path`, `is.blue.green`.

**Annotations used:** `@Service`, `@DependsOn`, `@Value`, `@PostConstruct`, `@PreDestroy` (all `javax.annotation.*`), and Lombok's `@Slf4j`.

**Things to note**
- The executor is created with `Executors` directly, not as a Spring bean. That's fine for one private thread, but the thread is named `pool-N-thread-1`, which is hard to spot in thread dumps. Pass a `ThreadFactory` with a name.
- `shutdown()` alone doesn't wait. The task finishes quickly, so that's acceptable here.
- Exceptions are caught *inside* the runnable. That matters because `execute()` would otherwise send them to the thread's default uncaught-exception handler, which only prints to stderr.

---

### 1B. `XmlGenerationService`: debounced job (anti-pattern)

**What it does**

An HTTP endpoint (`XmlGenerationController`) calls `generateXmlIfNotScheduled()`. If no run is already pending, the method:
- starts a thread that sleeps 5 minutes and then generates the `FINAL_TERMS`, `FINAL_TERMS_SHELL`, `KID` and `MIFID_PRIIP` files
- starts a **second** thread that also sleeps 5 minutes and then resets the `isScheduled` flag

```java
private AtomicBoolean isScheduled = new AtomicBoolean(false);

public synchronized void generateXmlIfNotScheduled() {
   if (!isScheduled.get()) {
      isScheduled.set(true);
      scheduleIssuance();               // new Thread: sleep 5m → generate files → finally isScheduled=false
      unscheduleIssuanceAfterFinish();  // new Thread: sleep 5m → finally isScheduled=false
   }
}
```

**Problems**
- It imports `ScheduledExecutorService` but uses raw `new Thread(...)` with `TimeUnit.MINUTES.sleep(5)`. Each sleeping thread holds a whole platform thread for 5 minutes.
- **There's a race.** The reset thread clears the flag after 5 minutes even if generation is still running, so a second request can start a concurrent generation.
- `synchronized` plus `AtomicBoolean` is redundant. `compareAndSet` does both jobs.
- Nothing cleans up the threads on shutdown.

**How to write it properly** (see §4.3 for the full template):

```java
private final ScheduledExecutorService scheduler =
      Executors.newSingleThreadScheduledExecutor(named("xml-generation"));
private final AtomicBoolean isScheduled = new AtomicBoolean(false);

public void generateXmlIfNotScheduled() {
   if (isScheduled.compareAndSet(false, true)) {        // atomic check-and-set, no synchronized needed
      scheduler.schedule(() -> {
         try {
            LocalDateTime cutoff = LocalDateTime.now();
            documentDomainService.generateFile(DocumentType.FINAL_TERMS, cutoff);
            // ...
         } catch (Exception e) {
            log.error("Failed to generate xml", e);
         } finally {
            isScheduled.set(false);                     // reset only after work truly finishes
         }
      }, 5, TimeUnit.MINUTES);
   } else {
      log.info("Intraday xml generation job is already scheduled");
   }
}

@PreDestroy
void stop() { scheduler.shutdownNow(); }
```

---

### 1C. `AwsFileDownloadService`: parallel zip (commented out, and broken)

The live code is sequential. A commented-out `zipFileKidFiles()` tried this:

```java
ExecutorService executorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
ZipOutputStream zos = new ZipOutputStream(new FileOutputStream(zipFilePath));
Files.walk(sourceFolder).filter(Files::isRegularFile).forEach(file ->
    CompletableFuture.runAsync(() -> {
        zos.putNextEntry(new ZipEntry(...));   // ❌ ZipOutputStream is NOT thread-safe
        Files.copy(file, zos);
        zos.closeEntry();
    }, executorService));
executorService.shutdown();
executorService.awaitTermination(Long.MAX_VALUE, TimeUnit.NANOSECONDS);
```

**Lesson:** parallelism only helps when the shared sink is thread-safe. If several threads write entries into one `ZipOutputStream`, the archive gets corrupted. The parallel part should be the **I/O-bound downloads** (`cloudStorageService.getDocumentByCloudId`). Zipping should stay on one thread:

```java
List<CompletableFuture<Path>> downloads = ids.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> download(id), ioPool))   // parallel S3 GETs
    .toList();
CompletableFuture.allOf(downloads.toArray(CompletableFuture[]::new)).join();
zipSequentially(downloads.stream().map(CompletableFuture::join).toList()); // single-threaded zip
```

---

## 2. mtfi-instrument-management

### 2D. `TaskSchedulerConfiguration`: one pool for all `@Scheduled` jobs

**Why it exists:** by default Spring's `@Scheduled` runs **every** job on a **single thread**. This service has about 20 cron jobs (`RegisterWarrantService`, `RedemptionProcessService`, `AutoWarrantDelistService`, `IntradayIssuanceAlertService`, the verification schedulers, and others). With one thread, a slow job would delay all the others, so the service provides its own `TaskScheduler` backed by a pool.

```java
@Configuration
public class TaskSchedulerConfiguration {

   @Bean
   public TaskScheduler getTaskScheduler(@Value("${task.scheduler.thread.count:10}") int threadPoolSize) {
      ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(threadPoolSize);
      return new ConcurrentTaskScheduler(scheduledExecutorService);   // adapts JDK executor → Spring TaskScheduler
   }
}
```

```java
@SpringBootApplication(exclude = {KafkaAutoConfiguration.class, JmsAutoConfiguration.class})
@Import(ApplicationConfiguration.class)
@EnableScheduling                                     // turns on @Scheduled processing
public class MTFInstrumentManagementSpringBootApplication extends SpringBootServletInitializer { ... }
```

**How Spring picks it up:** `@EnableScheduling` registers `ScheduledAnnotationBeanPostProcessor`. That processor looks for a single `TaskScheduler` bean (or one named `taskScheduler`) and runs every `@Scheduled` method on it. The same bean is also injected directly into services that schedule work dynamically (2E).

**Properties**
- `task.scheduler.thread.count`: pool size. It isn't set in any `runtime.properties`, so the default of **10** applies.
- Cron properties in `resource/src/main/properties/<env>/runtime.properties`, for example:
  ```properties
  redemption.process.cron=0 22 * * * *
  market.trading.hours.start.cron=0 38 0 * * ?
  ```
- Spring cron has **6 fields**: `sec min hour day-of-month month day-of-week`.

**Things to note**
- `ConcurrentTaskScheduler` doesn't shut down the executor you pass in. The pool's non-daemon threads survive context close, and Tomcat logs "thread leak" warnings on redeploy. The fix is `ThreadPoolTaskScheduler` (§4.2), which manages its own lifecycle and names its threads.
- This is Spring Boot 1.5.x (`org.springframework.boot.web.support.SpringBootServletInitializer`, deployed as a WAR).

---

### 2E. Dynamic cron scheduling with `TaskScheduler` + `ScheduledFuture`

**What it does:** the trigger times aren't known at compile time; they come from the DB. For example, each underlying has a `redemptionValTime` in its own exchange timezone. A fixed `@Scheduled` job rebuilds the set of dynamic jobs every day:

```java
@Service
@PropertySource("classpath:runtime.properties")
@RequiredArgsConstructor
public class RedemptionProcessService {

    private final TaskScheduler taskScheduler;                      // the bean from 2D
    private List<ScheduledFuture> scheduledFuture = new ArrayList<>();
    private Map<LocalTime, List<UnderlyingEntity>> valuationTimeMap = new ConcurrentHashMap<>();

    @PostConstruct                                                  // run once at startup …
    public void setUpMarketOpeningHourScheduler() { scheduleRedemptionProcessJob(); }

    @Scheduled(cron = "${redemption.process.cron}")                 // … and re-plan periodically
    public void scheduleRedemptionProcessJob() {
        // 1. cancel everything we scheduled last time
        scheduledFuture.forEach(s -> s.cancel(true));
        scheduledFuture = new ArrayList<>();

        // 2. load today's termination requests → underlyings → convert valuation time to server TZ
        // 3. group by LocalTime so one job fires per distinct time
        valuationTimeMap = activeUnderlyingList.stream()
              .filter(u -> u.getRedemptionValTime() != null)
              .map(this::convertEntityToSystemTimeZone)
              .collect(groupingBy(u -> u.getRedemptionValTime().toLocalTime()));

        // 4. schedule one cron job per distinct time and keep the handle
        valuationTimeMap.keySet().forEach(t ->
              scheduledFuture.add(taskScheduler.schedule(
                    () -> changeWarrantStateAndPublishImmediateTrigger(t),
                    new CronTrigger(registerWarrantService.getCronExp(t)))));
    }
}
```

The cron expression is built from a `LocalTime` in `RegisterWarrantService.getCronExp`:

```java
public String getCronExp(LocalTime time) {
   return "0 " + time.getMinute() + " " + time.getHour() + " * * ?";   // daily at HH:mm:00
}
```

`RegisterWarrantService.scheduleMarketTradingHourJob()` (`@Scheduled(cron = "${market.trading.hours.start.cron}")`) follows the same plan: cancel the old futures, then schedule new ones. It only does this when the instance is the ZooKeeper leader.

**Key ideas**
- `TaskScheduler.schedule(Runnable, Trigger)` returns a `ScheduledFuture`. **Keep the handle**, because it's the only way to cancel the job later.
- `cancel(true)` interrupts the job if it's mid-run.
- Re-planning on a cron prevents duplicate jobs. You need a leader check (or blue/green check) in a cluster so that only one node runs the jobs.
- Watch for these risks: `scheduledFuture` is a plain `ArrayList` that is reassigned without synchronisation, and the `@PostConstruct` run can overlap the first `@Scheduled` run. Use a `CopyOnWriteArrayList` (or `synchronized`) if two threads can re-plan at once.

---

### 2F. `UnderlyingReferencePriceServiceTest`: testing wait/notify (flaky)

The production code blocks a caller until an async AMQ callback arrives:

```java
public synchronized BigDecimal getReferencePrice(String symbol) {
   publishImmediateTrigger(...);
   wait(2000);                    // released by notify() or after 2s
   return referencePrice;
}
public synchronized void handleImmediateTriggerRequest(ImmediateTriggerAlert alert) {
   referencePrice = alert.getPrice().getValue();
   notify();
}
```

The test uses a 2-thread pool to play both sides:

```java
ExecutorService executorService = Executors.newFixedThreadPool(2);
executorService.submit(() -> gbpUsd.set(service.getReferencePrice("GBPUSD")));
executorService.submit(() -> service.handleImmediateTriggerRequest(alert));
executorService.awaitTermination(1000, TimeUnit.MILLISECONDS);   // ❌ no shutdown() first
assertEquals(new BigDecimal(999), gbpUsd.get());
```

**Problems**
- `awaitTermination` without `shutdown()` always waits the full 1 second. It never returns early.
- **There's a lost-notify race.** If the second task runs first, `notify()` fires before `wait()`. The getter then waits the full 2s, the test's 1s window has already closed, and the assertion fails.
- `wait()` isn't in a `while (condition)` loop, so spurious wakeups aren't handled.

**A better pattern** for "block until an async reply arrives" uses `CompletableFuture`:

```java
private final Map<String, CompletableFuture<BigDecimal>> pending = new ConcurrentHashMap<>();

public BigDecimal getReferencePrice(String symbol) throws Exception {
   CompletableFuture<BigDecimal> f = pending.computeIfAbsent(symbol, k -> new CompletableFuture<>());
   publishImmediateTrigger(...);
   try { return f.get(2, TimeUnit.SECONDS); } finally { pending.remove(symbol); }
}
public void handleImmediateTriggerRequest(ImmediateTriggerAlert a) {
   Optional.ofNullable(pending.get(symbolOf(a))).ifPresent(f -> f.complete(a.getPrice().getValue()));
}
```

The test then needs no threads at all: complete the future first, then call the getter.

---

## 3. payments-gateway

### 3G. `ExternalVerificationRequestsServiceImpl`: parallel fan-out with timeouts ⭐

**What it does**

The endpoint `GET /payments-gateway/api/external/client/{clientId}/verificationRequests?status=..&verificationType=BANK|CARD|APPLE_PAY` combines results from **3 downstream services** using Feign clients:

| Source | Feign client | Call |
|-|-|-|
| bank | `PaymentsApi` (`url=${payments.target.url}`) | `getExternalBankVerificationRequests` |
| card | `CardWithdrawalApi` | `getCardVerificationRequestsByStatus` |
| wallet | `WalletCardVerificationApi` | `getExternalWalletVerificationRequests` |

Calling them in order would cost `t_bank + t_card + t_wallet`. Calling them **in parallel** costs about `max(t_bank, t_card, t_wallet)`. Each source **fails independently**: a timeout or error in one source marks it `timeout`/`error` in `meta.sources`, and the other sources still return their data (graceful degradation).

**Code (actual, trimmed)**

```java
@Slf4j
@Service
public class ExternalVerificationRequestsServiceImpl implements ExternalVerificationRequestsService {

    private static final long FETCH_TIMEOUT_SECONDS = 30;

    private final PaymentsApi paymentsApi;
    private final CardWithdrawalApi cardWithdrawalApi;
    private final WalletCardVerificationApi walletCardVerificationApi;
    private final ExecutorService asyncExecutor;
    private final long fetchTimeoutSeconds;

    // 1) Constructor Spring uses — builds its own bounded, daemon, named pool
    @Autowired
    public ExternalVerificationRequestsServiceImpl(PaymentsApi p, CardWithdrawalApi c, WalletCardVerificationApi w) {
        this(p, c, w, Executors.newFixedThreadPool(10, r -> {
            Thread t = new Thread(r, "verification-fetch");
            t.setDaemon(true);                          // never blocks JVM shutdown
            return t;
        }), FETCH_TIMEOUT_SECONDS);
    }

    // 2) Package-private constructors — tests inject a mock/inline executor and a short timeout
    ExternalVerificationRequestsServiceImpl(PaymentsApi p, CardWithdrawalApi c, WalletCardVerificationApi w,
                                            ExecutorService asyncExecutor) { this(p, c, w, asyncExecutor, FETCH_TIMEOUT_SECONDS); }

    ExternalVerificationRequestsServiceImpl(PaymentsApi p, CardWithdrawalApi c, WalletCardVerificationApi w,
                                            ExecutorService asyncExecutor, long fetchTimeoutSeconds) { ... }

    @PreDestroy                                          // jakarta.annotation.PreDestroy (Boot 3)
    public void shutdown() { asyncExecutor.shutdown(); }

    @Override
    public ExternalVerificationRequestsResponseDTO getVerificationRequests(String clientId,
                                                                           List<String> statuses,
                                                                           List<String> verificationTypes) {
        Map<String, String> sourceStatuses = new HashMap<>();   // only touched by the calling thread
        List<String> effectiveStatuses = CollectionUtils.isEmpty(statuses) ? null : statuses;

        // FAN-OUT: start every requested call immediately on the pool
        CompletableFuture<List<BankVerificationRequestDTO>> bankFuture =
              isTypeRequested(verificationTypes, "BANK")
                    ? CompletableFuture.supplyAsync(() -> paymentsApi.getExternalBankVerificationRequests(clientId, effectiveStatuses), asyncExecutor)
                    : null;
        CompletableFuture<List<CardVerificationRequestDTO>> cardFuture = ...;
        CompletableFuture<List<ExternalWalletVerificationReqResDTO>> walletFuture = ...;

        // FAN-IN: wait for each with a timeout; failures are isolated per source
        Optional<List<BankVerificationRequestDTO>> bankResult =
              bankFuture != null ? awaitVerificationResult(bankFuture, "bank", clientId, sourceStatuses) : null;
        // ... card, wallet ...

        return ExternalVerificationRequestsResponseDTO.builder()
              .verificationRequests(VerificationRequests.builder().bank(...).card(...).wallet(...).build())
              .meta(Meta.builder().sources(sourceStatuses).build())   // {"bank":"success","card":"timeout",...}
              .build();
    }

    private <T> Optional<List<T>> awaitVerificationResult(CompletableFuture<List<T>> future, String source,
                                                          String clientId, Map<String, String> sourceStatuses) {
        try {
            List<T> result = future.get(fetchTimeoutSeconds, TimeUnit.SECONDS);
            sourceStatuses.put(source, "success");
            return Optional.of(result);
        } catch (TimeoutException e) {
            future.cancel(false);   // NB: does NOT abort the in-flight Feign HTTP call
            sourceStatuses.put(source, "timeout");
            return Optional.empty();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();          // ✅ restore interrupt flag
            sourceStatuses.put(source, "error");
            return Optional.empty();
        } catch (ExecutionException e) {                 // Feign exception thrown inside supplyAsync
            sourceStatuses.put(source, "error");
            return Optional.empty();
        }
    }
}
```

**Supporting config** (`resource/src/main/resources/properties/common/application.yml`)

```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: false
      client:
        config:
          default:
            connectTimeout: 10000   # ms
            readTimeout: 30000      # ms — matches FETCH_TIMEOUT_SECONDS = 30
server:
  servlet:
    context-path: /payments-gateway
  port: 8080
```

Stack: Java 17, Spring Boot 3.5.16, Spring Cloud 2025.0.3, IG `ig-feign-spring-boot-starter` 1.0.39, with Feign clients declared through `@FeignClientDecorator(name = "PaymentsApi", url = "${payments.target.url}")`.

**Why it's built this way**
- **Bounded pool (10).** A flood of requests can't create unlimited threads. Note that `newFixedThreadPool` has an **unbounded queue**, though: under sustained overload, tasks queue up and wait toward the 30s timeout instead of being rejected.
- **Daemon and named threads.** They don't block shutdown, and you can find them in a thread dump by searching for `verification-fetch`.
- **The executor is passed explicitly to `supplyAsync`.** Without it, `supplyAsync` uses `ForkJoinPool.commonPool()`, which is shared JVM-wide and sized to CPU count. Blocking HTTP calls on it starve parallel streams and other users.
- **A test seam via constructor injection.** Tests pass a mock executor, so they can run tasks inline and stay deterministic.
- **Each `ExecutionException` is caught separately**, which gives partial results instead of an all-or-nothing 500.

**Things to improve**
1. **Worst-case latency is 90s, not 30s.** The `get(30s)` waits happen one after another. If all three calls hang, bank times out at about 30s, then card's wait starts and runs to about 60s, then wallet to about 90s. In practice Feign's `readTimeout` (30s) plus `connectTimeout` (10s) cap each call around 40s, but the clean fix is a **shared deadline**:
   ```java
   long deadline = System.nanoTime() + TimeUnit.SECONDS.toNanos(fetchTimeoutSeconds);
   long remaining = Math.max(0, deadline - System.nanoTime());
   future.get(remaining, TimeUnit.NANOSECONDS);
   ```
   Alternatively, use `CompletableFuture.orTimeout(...)` / `completeOnTimeout(...)` (Java 9+) on each future, then `allOf(...).join()`.
2. `future.cancel(false)` only marks the future. The HTTP call keeps running and keeps a pool thread busy. The log message in the code says so explicitly. The Feign `readTimeout` is what actually frees the thread.
3. All threads share the name `verification-fetch`. Add a counter (`verification-fetch-1`, `-2`, …).
4. `shutdown()` doesn't wait. Use `shutdown()`, then `awaitTermination(5s)`, then `shutdownNow()` for a graceful drain.
5. MDC/trace context isn't propagated to pool threads. If logs need `traceId`, wrap the executor (Micrometer `ContextExecutorService.wrap(...)`) or use Spring's `ThreadPoolTaskExecutor` with a `TaskDecorator`.

**How it's tested** (`ExternalVerificationRequestsServiceImplTest`)

```java
@ExtendWith(MockitoExtension.class)
class ExternalVerificationRequestsServiceImplTest {

    @Mock PaymentsApi paymentsApi;
    @Mock CardWithdrawalApi cardWithdrawalApi;
    @Mock WalletCardVerificationApi walletCardVerificationApi;
    @Mock ExecutorService asyncExecutor;

    ExternalVerificationRequestsServiceImpl service;

    @BeforeEach
    void setUp() {
        // CompletableFuture.supplyAsync(..., executor) calls executor.execute(Runnable).
        // Make the mock run it inline → fully synchronous, deterministic tests.
        lenient().doAnswer(inv -> { inv.getArgument(0, Runnable.class).run(); return null; })
                 .when(asyncExecutor).execute(any());
        service = new ExternalVerificationRequestsServiceImpl(paymentsApi, cardWithdrawalApi,
                                                              walletCardVerificationApi, asyncExecutor);
    }

    @Test
    void returnsEmptyWithErrorStatus_whenFeignExceptionOccurs() {
        when(paymentsApi.getExternalBankVerificationRequests(eq("108313203"), isNull()))
              .thenThrow(FeignException.errorStatus("test", /* Response */ ...));
        var response = service.getVerificationRequests("108313203", null, List.of("BANK"));
        assertThat(response.getMeta().getSources().get("bank")).isEqualTo("error");
    }

    @Test
    void shutdown_shouldDelegateToExecutorService() {
        service.shutdown();
        verify(asyncExecutor).shutdown();
    }
}
```

Instead of a mock you can use `Runnable::run` as the executor (`Executor` is a functional interface). Use the 5-arg constructor with `fetchTimeoutSeconds = 1` plus a real pool to test the timeout branch.

---

### 3H. `IGClusterDetails`: periodic file polling

This class is copied into about 17 payments repos. Its constructor starts a daemon single-thread scheduler that re-reads the light/dark state file every second:

```java
public IGClusterDetails() {
   ...
   String liveDarkStateFile = System.getProperty("light.dark.state.file", "OVlivedark");
   getExecutorService().scheduleAtFixedRate(new LightDarkStateUpdateRunnable(liveDarkStateFile),
                                            0, 1, TimeUnit.SECONDS);
}

private static ScheduledExecutorService getExecutorService() {
   return Executors.newSingleThreadScheduledExecutor(runnable -> {
      Thread thread = Executors.defaultThreadFactory().newThread(runnable);
      thread.setDaemon(true);
      return thread;
   });
}

private class LightDarkStateUpdateRunnable implements Runnable {
   private Date lastLoaded = new Date(0);
   public void run() {
      File f = new File(lightDarkStateFile);
      if (f.exists() && new Date(f.lastModified()).after(lastLoaded)) {   // only reload when file changed
         lightDarkState = new Scanner(f).useDelimiter("\\Z").next();
         System.setProperty("light.dark.state", lightDarkState);
         lastLoaded = new Date();
      }
   }
}
```

**JVM system properties it reads:** `light.dark.state.file`, `spring.profiles.active` (colour), `java.rmi.server.hostname`, `catalina.instance`, `site`.

**Things to note**
- Every `new IGClusterDetails()` starts **another** thread, and nothing ever shuts it down. In payments-gateway it's constructed in `ServiceModeConverter`, and the library version is constructed separately in `GoldenSignalsConfiguration`. Make it a singleton bean with `@PreDestroy`.
- `lightDarkState` is written by the poller thread and read by request threads without `volatile`, so request threads may see a stale value. Mark the field `volatile`.
- If `run()` throws an **unchecked** exception, `scheduleAtFixedRate` **silently cancels all future runs**. Always wrap the body in `try { … } catch (Exception e) { log… }`. The code catches only `IOException` here.
- `new Scanner(f)` is never closed, so it leaks a file handle on every reload. Use try-with-resources or `Files.readString(path)`.

---

## 4. Recreate it yourself: the guide

### 4.1 Dependencies (Maven, Spring Boot 3 / Java 17+)

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.5.x</version>
</parent>

<properties>
  <java.version>17</java.version>          <!-- 21 if you want virtual threads -->
  <spring-cloud.version>2025.0.x</spring-cloud.version>
</properties>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type><scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
  <dependency><groupId>org.springframework.cloud</groupId><artifactId>spring-cloud-starter-openfeign</artifactId></dependency>   <!-- pattern G -->
  <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId><optional>true</optional></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
  <dependency><groupId>org.awaitility</groupId><artifactId>awaitility</artifactId><scope>test</scope></dependency>          <!-- async assertions -->
  <!-- pattern A only: -->
  <dependency><groupId>org.apache.curator</groupId><artifactId>curator-recipes</artifactId><version>5.x</version></dependency>
  <dependency><groupId>org.springframework.kafka</groupId><artifactId>spring-kafka</artifactId></dependency>
</dependencies>
```

> Boot 2.x / 1.5.x (the mtfi repos) use `javax.annotation.PostConstruct/PreDestroy`. Boot 3 uses `jakarta.annotation.*`.

### 4.2 Executors as Spring beans (preferred over `Executors.*` in fields)

```java
@Configuration
@EnableScheduling     // for @Scheduled (pattern D/E)
@EnableAsync          // only if you want @Async
public class ExecutorConfig {

    /** Pattern G: bounded I/O pool for downstream fan-out. */
    @Bean(name = "verificationExecutor", destroyMethod = "shutdown")
    public ExecutorService verificationExecutor(@Value("${app.verification.pool-size:10}") int size,
                                                @Value("${app.verification.queue-capacity:100}") int queue) {
        AtomicInteger n = new AtomicInteger();
        return new ThreadPoolExecutor(
                size, size, 0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(queue),                     // BOUNDED queue, unlike newFixedThreadPool
                r -> { Thread t = new Thread(r, "verification-fetch-" + n.incrementAndGet()); t.setDaemon(true); return t; },
                new ThreadPoolExecutor.CallerRunsPolicy());          // back-pressure instead of OOM
    }

    /** Pattern D: pool for every @Scheduled method + dynamic schedules. Spring manages shutdown. */
    @Bean
    public ThreadPoolTaskScheduler taskScheduler(@Value("${task.scheduler.thread.count:10}") int size) {
        ThreadPoolTaskScheduler s = new ThreadPoolTaskScheduler();
        s.setPoolSize(size);
        s.setThreadNamePrefix("sched-");
        s.setWaitForTasksToCompleteOnShutdown(true);
        s.setAwaitTerminationSeconds(30);
        s.setErrorHandler(t -> LoggerFactory.getLogger("scheduler").error("Scheduled task failed", t));
        return s;
    }

    /** Spring-flavoured alternative to a raw ExecutorService; supports TaskDecorator for MDC. */
    @Bean
    public ThreadPoolTaskExecutor ioTaskExecutor() {
        ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
        e.setCorePoolSize(10);
        e.setMaxPoolSize(10);
        e.setQueueCapacity(100);
        e.setThreadNamePrefix("io-");
        e.setTaskDecorator(r -> {                               // copy MDC (traceId etc.) to worker thread
            Map<String, String> mdc = MDC.getCopyOfContextMap();
            return () -> { if (mdc != null) MDC.setContextMap(mdc); try { r.run(); } finally { MDC.clear(); } };
        });
        e.initialize();
        return e;
    }
}
```

**Zero-code alternative (Boot 2.1+).** Spring Boot auto-configures the `@Scheduled`/`@Async` pools from these properties:

```yaml
spring:
  task:
    scheduling:
      pool:
        size: 10                    # replaces TaskSchedulerConfiguration entirely
      thread-name-prefix: sched-
      shutdown:
        await-termination: true
        await-termination-period: 30s
    execution:
      pool:
        core-size: 10
        max-size: 10
        queue-capacity: 100
      thread-name-prefix: io-
  threads:
    virtual:
      enabled: true                 # Boot 3.2+ & Java 21: virtual threads for @Async / web / scheduling
```

### 4.3 Templates per pattern

**Pattern A: do blocking startup work in the background**

```java
@Slf4j
@Service
@DependsOn({"containerA", "containerB"})
public class FlipService {

    private final ExecutorService executor =
          Executors.newSingleThreadExecutor(r -> new Thread(r, "flip-service"));
    private final CuratorFramework curator;
    private final TreeCache cache;
    private final List<MessageListenerContainer> containers;
    private final String path;
    private final String myColour;

    public FlipService(CuratorFramework curator, TreeCache cache, List<MessageListenerContainer> containers,
                       @Value("${zookeeper.root.path}") String path, @Value("${deployment.colour}") String myColour) { ... }

    @PostConstruct
    void start() {
        executor.execute(() -> {
            try {
                apply(isLight(curator.getData().forPath(path)));
                cache.getListenable().addListener((c, e) -> {
                    if (e.getType() == TreeCacheEvent.Type.NODE_ADDED || e.getType() == TreeCacheEvent.Type.NODE_UPDATED) {
                        apply(isLight(e.getData().getData()));
                    }
                });
            } catch (Exception e) {
                log.error("FlipService start failed", e);
            }
        });
    }

    private void apply(boolean light) { containers.forEach(c -> { if (light) c.start(); else c.stop(); }); }
    private boolean isLight(byte[] d) { return d != null && myColour.equals(new String(d, StandardCharsets.UTF_8)); }

    @PreDestroy
    void stop() throws InterruptedException {
        executor.shutdown();
        if (!executor.awaitTermination(5, TimeUnit.SECONDS)) executor.shutdownNow();
    }
}
```
Containers: `factory.createContainer(topic)` followed by `container.setAutoStartup(false)`. TreeCache bean: `@Bean(initMethod = "start", destroyMethod = "close") TreeCache treeCache(CuratorFramework c) { return TreeCache.newBuilder(c, path).build(); }`.

**Pattern B (fixed): debounced delayed job.** See the code in §1B.

**Pattern D/E: dynamic cron jobs**

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class DynamicJobPlanner {

    private final TaskScheduler taskScheduler;
    private final JobRepository repo;
    private final List<ScheduledFuture<?>> handles = new CopyOnWriteArrayList<>();

    @EventListener(ApplicationReadyEvent.class)       // safer than @PostConstruct: context fully ready
    public void onStartup() { replan(); }

    @Scheduled(cron = "${planner.replan.cron:0 0 0 * * *}", zone = "${planner.zone:Europe/London}")
    public synchronized void replan() {
        handles.forEach(h -> h.cancel(false));         // false = let a running job finish
        handles.clear();
        repo.findTimesForToday().stream().distinct().forEach(t ->
              handles.add(taskScheduler.schedule(() -> runFor(t),
                    new CronTrigger(String.format("0 %d %d * * *", t.getMinute(), t.getHour()),
                                    ZoneId.of("Europe/London")))));
        log.info("Planned {} jobs", handles.size());
    }

    private void runFor(LocalTime t) {
        try { /* work */ } catch (Exception e) { log.error("Job {} failed", t, e); }
    }
}
```
For a one-shot job, use `taskScheduler.schedule(task, Instant)` instead of a `CronTrigger` when it should fire once rather than daily.

**Pattern G: parallel fan-out with an overall deadline**

```java
@Slf4j
@Service
public class AggregatorService {

    private final PaymentsApi bankApi;
    private final CardApi cardApi;
    private final WalletApi walletApi;
    private final Executor executor;
    private final Duration timeout;

    public AggregatorService(PaymentsApi bankApi, CardApi cardApi, WalletApi walletApi,
                             @Qualifier("verificationExecutor") Executor executor,
                             @Value("${app.verification.timeout:30s}") Duration timeout) { ... }

    public Response aggregate(String clientId, Set<String> types) {
        Map<String, String> status = new ConcurrentHashMap<>();
        var bank   = types.contains("BANK")   ? call("bank",   () -> bankApi.get(clientId),   status) : null;
        var card   = types.contains("CARD")   ? call("card",   () -> cardApi.get(clientId),   status) : null;
        var wallet = types.contains("WALLET") ? call("wallet", () -> walletApi.get(clientId), status) : null;

        CompletableFuture.allOf(Stream.of(bank, card, wallet).filter(Objects::nonNull)
                                      .toArray(CompletableFuture[]::new)).join();   // never throws: each future is self-healing
        return new Response(join(bank), join(card), join(wallet), status);
    }

    private <T> CompletableFuture<List<T>> call(String src, Supplier<List<T>> s, Map<String, String> status) {
        return CompletableFuture.supplyAsync(s, executor)
              .orTimeout(timeout.toMillis(), TimeUnit.MILLISECONDS)            // all start together → shared deadline
              .handle((res, ex) -> {
                  if (ex == null) { status.put(src, "success"); return res; }
                  Throwable cause = ex instanceof CompletionException ? ex.getCause() : ex;
                  status.put(src, cause instanceof TimeoutException ? "timeout" : "error");
                  log.error("source={} failed", src, cause);
                  return List.<T>of();
              });
    }

    private static <T> List<T> join(CompletableFuture<List<T>> f) { return f == null ? null : f.join(); }
}
```
```yaml
app:
  verification:
    pool-size: 10
    queue-capacity: 100
    timeout: 30s
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connectTimeout: 10000
            readTimeout: 30000        # keep ≤ app.verification.timeout so threads are actually freed
```
Enable Feign with `@EnableFeignClients` on the application class and declare clients with `@FeignClient(name = "PaymentsApi", url = "${payments.target.url}")`.

**Pattern H: safe periodic poller**

```java
@Slf4j
@Component
public class LightDarkStatePoller {

    private final ScheduledExecutorService ses =
          Executors.newSingleThreadScheduledExecutor(r -> { Thread t = new Thread(r, "light-dark-poller"); t.setDaemon(true); return t; });
    private final Path file;
    private volatile String state = "UNKNOWN";             // visible to request threads
    private volatile long lastModified = 0;

    public LightDarkStatePoller(@Value("${light.dark.state.file:OVlivedark}") String file) { this.file = Path.of(file); }

    @PostConstruct
    void start() { ses.scheduleWithFixedDelay(this::poll, 0, 1, TimeUnit.SECONDS); }

    private void poll() {
        try {                                              // catch-all: an escaped exception kills the schedule
            if (Files.exists(file)) {
                long m = Files.getLastModifiedTime(file).toMillis();
                if (m > lastModified) {
                    state = Files.readString(file).strip();
                    lastModified = m;
                    log.info("Light/Dark state now {}", state);
                }
            }
        } catch (Exception e) {
            log.error("Light/dark poll failed", e);
        }
    }

    public String state() { return state; }

    @PreDestroy
    void stop() { ses.shutdownNow(); }
}
```
In a Spring app, `@Scheduled(fixedDelay = 1000)` on a method does the same job with no executor code.

### 4.4 Testing recipes

| Goal | Technique |
|-|-|
| Deterministic unit tests of async code | Inject `Runnable::run` (or a Mockito mock that runs `execute()` args inline, as in payments-gateway) |
| Timeout branch | Real pool + a supplier that `Thread.sleep`s past a tiny `fetchTimeoutSeconds` / `Duration.ofMillis(50)` |
| Failure branch | `when(api.call()).thenThrow(FeignException...)` → assert `status == "error"` |
| Dynamic scheduling | Mock `TaskScheduler`, `ArgumentCaptor<Trigger>`, assert cron string; assert old `ScheduledFuture.cancel()` called |
| Real concurrency | `CountDownLatch` to line up threads, `Awaitility.await().atMost(2, SECONDS).until(...)` for assertions |
| Shutdown | `service.shutdown(); verify(executor).shutdown();` |
| Always | Call `shutdown()` **before** `awaitTermination()` in tests (2F got this wrong) |

### 4.5 Checklist (all the issues above, condensed)

- [ ] **Never** do blocking I/O on `ForkJoinPool.commonPool()`. Always pass an executor to `supplyAsync`/`runAsync`.
- [ ] Bound both threads **and** queue; pick a rejection policy (`CallerRunsPolicy` for back-pressure).
- [ ] Name threads (`ThreadFactory` / `setThreadNamePrefix`), using unique names with a counter.
- [ ] Daemon threads for background helpers; Spring-managed lifecycle for anything doing real work.
- [ ] Shut down every executor you create: `@PreDestroy` or `@Bean(destroyMethod="shutdown")`, using shutdown, then awaitTermination, then shutdownNow.
- [ ] Wrap scheduled task bodies in try/catch, because an uncaught exception silently cancels `scheduleAtFixedRate`.
- [ ] Use `volatile`, atomics, or concurrent collections for state shared across threads.
- [ ] Use `compareAndSet` rather than `get()` followed by `set()`.
- [ ] Keep `ScheduledFuture` handles if jobs must be cancelled or re-planned.
- [ ] Timeouts: use a shared deadline (`orTimeout` / remaining-nanos), and align them with the HTTP client's read timeout.
- [ ] Restore the interrupt flag on `InterruptedException`.
- [ ] Never share non-thread-safe sinks (e.g. `ZipOutputStream`, `SimpleDateFormat`, `HashMap`) across tasks.
- [ ] Propagate MDC/trace context with `TaskDecorator` or Micrometer context propagation.
- [ ] Clusters: guard scheduled jobs with a leader election or blue/green check (ZooKeeper here; ShedLock is an alternative).
- [ ] Prefer `@Scheduled` + `spring.task.scheduling.pool.size` over a hand-built `TaskSchedulerConfiguration`.

---

### Source references

- `_non-clinet-tech/mtfi-document-service/integration/src/main/java/com/iggroup/mtfi/document/service/FlipService.java`
- `_non-clinet-tech/mtfi-document-service/integration/src/main/java/com/iggroup/mtfi/document/service/configuration/{KafkaConfiguration,LeadershipConfiguration}.java`
- `_non-clinet-tech/mtfi-document-service/domain/src/main/java/com/iggroup/mtfi/document/service/service/XmlGenerationService.java`
- `_non-clinet-tech/mtfi-document-service/integration/src/main/java/com/iggroup/mtfi/document/service/service/gateway/AwsFileDownloadService.java`
- `_non-clinet-tech/mtfi-instrument-management/integration/src/main/java/com/raydius/instrument/management/configuration/TaskSchedulerConfiguration.java`
- `_non-clinet-tech/mtfi-instrument-management/integration/src/main/java/com/raydius/instrument/management/redemption/RedemptionProcessService.java`
- `_non-clinet-tech/mtfi-instrument-management/integration/src/main/java/com/raydius/instrument/management/schduler/RegisterWarrantService.java`
- `_non-clinet-tech/mtfi-instrument-management/integration/src/test/java/com/raydius/instrument/management/referenceprice/UnderlyingReferencePriceServiceTest.java`
- `_non-clinet-tech/mtfi-instrument-management/resource/src/main/properties/<env>/runtime.properties`
- `payments/payments-gateway/integration/src/main/java/com/iggroup/wt/payments/gateway/web/service/ExternalVerificationRequestsServiceImpl.java`
- `payments/payments-gateway/integration/src/test/java/com/iggroup/wt/payments/gateway/web/service/ExternalVerificationRequestsServiceImplTest.java`
- `payments/payments-gateway/integration/src/main/java/com/iggroup/wt/payments/gateway/web/controllers/ExternalVerificationRequestsController.java`
- `payments/payments-gateway/integration/src/main/java/com/iggroup/wt/payments/gateway/observability/logging/IGClusterDetails.java`
- `payments/payments-gateway/resource/src/main/resources/properties/common/application.yml`
- JDK docs: `java.util.concurrent.ExecutorService`, `ScheduledExecutorService`, `CompletableFuture` (Java 17)
- Spring docs: "Task Execution and Scheduling" (Spring Framework 6), Spring Boot `spring.task.*` properties
