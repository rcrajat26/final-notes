## Card-Payments EOD → Data Warehouse: Chunked Parallel Send

### The scenario, modeled
```
DB (Card Payments) → Orchestrator → chunk into batches → ThreadPool → transform + send → DW
                                                                ↓
                                                    collect Futures → build final report
```

### The domain objects
```java
record CardPayment(String txnId, String cardLast4, double amount, String currency, Instant txnTime) {}

// Result of sending one chunk to DW
record ChunkResult(int chunkId, List<CardPayment> chunkData, boolean success, int recordCount, String errorMessage, int attemptNumber) {}
```
We keep chunkData in the result so a retry doesn't need to re-fetch from the DB — it just resubmits the same in-memory batch.

### The task — `Callable`, not `Runnable`
We use `Callable<ChunkResult>` instead of `Runnable` because we need a return value (success/failure, count) and want to propagate exceptions back to the orchestrator instead of losing them in a background thread.

```java
class ChunkSendTask implements Callable<ChunkResult> {
    private final int chunkId;
    private final List<CardPayment> chunk;
    private final DWClient dwClient;

    ChunkSendTask(int chunkId, List<CardPayment> chunk, DWClient dwClient) {
        this.chunkId = chunkId;
        this.chunk = chunk;
        this.dwClient = dwClient;
    }

    @Override
    public ChunkResult call() {
        try {
            // 1. Transform to DW's required format (e.g., Avro/Parquet/JSON schema)
            String payload = DWFormatter.toDWFormat(chunk);

            // 2. Send (HTTP POST / SFTP / Kafka publish — abstracted behind DWClient)
            dwClient.send(payload, chunkId);

            return new ChunkResult(chunkId, true, chunk.size(), null);
        } catch (Exception e) {
            // Never let the exception die silently inside the pool thread —
            // capture it in the result instead of throwing
            return new ChunkResult(chunkId, false, chunk.size(), e.getMessage());
        }
    }
}
```

### The orchestrator

```java
class CardPaymentEodOrchestrator {

    private static final int CHUNK_SIZE = 5_000;

    private final CardPaymentRepository repository; // DB access
    private final DWClient dwClient;

    private final ThreadPoolExecutor executor = new ThreadPoolExecutor(
        4,                                          // corePoolSize — steady-state workers
        8,                                          // maximumPoolSize — burst capacity
        30L, TimeUnit.SECONDS,                      // idle extra threads die after 30s
        new ArrayBlockingQueue<>(50),                // BOUNDED — avoids OOM on huge EOD runs
        new NamedThreadFactory("dw-sender"),         // custom factory for readable thread names
        new ThreadPoolExecutor.CallerRunsPolicy()    // backpressure: caller thread sends if pool is saturated
    );

    CardPaymentEodOrchestrator(CardPaymentRepository repository, DWClient dwClient) {
        this.repository = repository;
        this.dwClient = dwClient;
    }

    EodReport runEodSend(LocalDate businessDate) {
        List<CardPayment> allPayments = repository.fetchCardPaymentsForDate(businessDate);
        List<List<CardPayment>> chunks = chunkList(allPayments, CHUNK_SIZE);

        List<Future<ChunkResult>> futures = new ArrayList<>();
        for (int i = 0; i < chunks.size(); i++) {
            ChunkSendTask task = new ChunkSendTask(i, chunks.get(i), dwClient);
            futures.add(executor.submit(task));   // submit() -> returns Future immediately, doesn't block
        }

        // Collect results — this is where we actually wait for completion
        List<ChunkResult> results = new ArrayList<>();
        for (Future<ChunkResult> future : futures) {
            try {
                // timeout so one stuck chunk can't hang the whole EOD job forever
                results.add(future.get(2, TimeUnit.MINUTES));
            } catch (TimeoutException e) {
                future.cancel(true); // best-effort interrupt
                results.add(new ChunkResult(-1, false, 0, "Timed out"));
            } catch (ExecutionException | InterruptedException e) {
                results.add(new ChunkResult(-1, false, 0, e.getMessage()));
            }
        }

        shutdownGracefully();
        return buildReport(businessDate, results);
    }

    private void shutdownGracefully() {
        executor.shutdown(); // stop accepting new tasks, let submitted ones finish
        try {
            if (!executor.awaitTermination(5, TimeUnit.MINUTES)) {
                executor.shutdownNow(); // force-interrupt anything still running
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }
    }

    private EodReport buildReport(LocalDate date, List<ChunkResult> results) {
        long success = results.stream().filter(ChunkResult::success).count();
        long failed = results.size() - success;
        int totalRecords = results.stream().mapToInt(ChunkResult::recordCount).sum();
        List<String> errors = results.stream()
                .filter(r -> !r.success())
                .map(r -> "Chunk " + r.chunkId() + ": " + r.errorMessage())
                .toList();

        return new EodReport(date, results.size(), success, failed, totalRecords, errors);
    }

    private static <T> List<List<T>> chunkList(List<T> list, int size) {
        List<List<T>> chunks = new ArrayList<>();
        for (int i = 0; i < list.size(); i += size) {
            chunks.add(list.subList(i, Math.min(i + size, list.size())));
        }
        return chunks;
    }
}
```

### Custom ThreadFactory (why bother?)
By default, pooled threads get generic names like pool-1-thread-3 — useless in logs during an incident. A named factory fixes that:
```java
class NamedThreadFactory implements ThreadFactory {
    private final AtomicInteger counter = new AtomicInteger(1);
    private final String prefix;

    NamedThreadFactory(String prefix) { this.prefix = prefix; }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, prefix + "-" + counter.getAndIncrement());
        t.setDaemon(false); // JVM shouldn't exit while EOD send is in progress
        return t;
    }
}
```

Now your logs read dw-sender-3: sending chunk 7 instead of pool-1-thread-3: ... — directly traceable to this job.

### Report model

```java
record EodReport(
    LocalDate businessDate,
    int totalChunks,
    long successfulChunks,
    long failedChunks,
    int totalRecordsSent,
    List<String> errors
) {}
```

## Design decisions worth calling out

| Decision | Why |
|---|---|
| `Callable<ChunkResult>` over `Runnable` | Need return value + need to know per-chunk success/failure for the final report |
| `submit()` not `execute()` | `submit()` returns a `Future`; `execute()` (from plain `Runnable`) returns nothing — you'd have no way to know if/when a chunk finished or failed |
| Bounded `ArrayBlockingQueue` | An EOD run could be huge (millions of rows); an unbounded queue risks OOM if DB reads outpace DW sends |
| `CallerRunsPolicy` | When the pool+queue are saturated, the *submitting* thread does the send itself — natural backpressure, slows chunk production instead of dropping/crashing |
| `future.get(timeout)` | One hung network call to DW shouldn't block the whole EOD job indefinitely |
| Catching exceptions **inside** `call()` | If a task throws instead, the exception is wrapped in `ExecutionException` and only surfaces at `future.get()` — easy to lose track of which chunk failed if you're not careful. Capturing it in `ChunkResult` keeps success/failure explicit and per-chunk |
| Fixed core/max at 4/8 | Card-payment sends are likely I/O-bound (network to DW) — a modest pool with headroom for bursts, not CPU-core-bound sizing |

## What this buys you operationally

- **Partial failure visibility** — if 2 of 50 chunks fail, you get a report naming exactly which ones, instead of an all-or-nothing job
- **Retry-ability** — since `EodReport.errors` tells you which chunk IDs failed, a retry job can re-fetch and resend just those chunks rather than the whole day's data
- **Reusable pool if extended** — this `ThreadPoolExecutor` could be a long-lived `@Bean`/singleton in a Spring app, shared across Card, UPI, NetBanking payment-type orchestrators, rather than creating a fresh pool per job

---

## Retry-aware orchestrator method
### Orchestrator
```java
class CardPaymentEodOrchestrator {

    private static final int CHUNK_SIZE = 5_000;
    private static final int MAX_RETRIES = 3;
    private static final long BASE_BACKOFF_MS = 1000;

    private final CardPaymentRepository repository;
    private final DWClient dwClient;
    private final ThreadPoolExecutor executor = new ThreadPoolExecutor(
        4, 8, 30L, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(50),
        new NamedThreadFactory("dw-sender"),
        new ThreadPoolExecutor.CallerRunsPolicy()
    );

    EodReport runEodSend(LocalDate businessDate) {
        List<CardPayment> allPayments = repository.fetchCardPaymentsForDate(businessDate);
        List<List<CardPayment>> chunks = chunkList(allPayments, CHUNK_SIZE);

        // Attempt 1: submit everything
        Map<Integer, ChunkResult> resultsByChunkId = submitAndCollect(chunks, 1);

        // Retry loop: only resubmit failures
        int attempt = 1;
        while (attempt < MAX_RETRIES) {
            List<Map.Entry<Integer, ChunkResult>> failed = resultsByChunkId.entrySet().stream()
                    .filter(e -> !e.getValue().success())
                    .toList();

            if (failed.isEmpty()) break; // everything succeeded, stop early

            attempt++;
            sleepWithBackoff(attempt);

            System.out.printf("Retry attempt %d for %d failed chunks%n", attempt, failed.size());

            // Resubmit only the failed chunks' original data
            List<List<CardPayment>> retryBatch = failed.stream()
                    .map(e -> e.getValue().chunkData())
                    .toList();
            List<Integer> retryChunkIds = failed.stream().map(Map.Entry::getKey).toList();

            Map<Integer, ChunkResult> retryResults = submitAndCollectWithIds(retryBatch, retryChunkIds, attempt);

            // Merge retry results back over the old failed entries
            resultsByChunkId.putAll(retryResults);
        }

        shutdownGracefully();
        return buildReport(businessDate, new ArrayList<>(resultsByChunkId.values()), attempt);
    }

    /** First-pass submit: chunk index == chunk id */
    private Map<Integer, ChunkResult> submitAndCollect(List<List<CardPayment>> chunks, int attemptNumber) {
        List<Integer> ids = IntStream.range(0, chunks.size()).boxed().toList();
        return submitAndCollectWithIds(chunks, ids, attemptNumber);
    }

    /** Generic submit: lets retries reuse original chunk IDs instead of renumbering */
    private Map<Integer, ChunkResult> submitAndCollectWithIds(
            List<List<CardPayment>> chunks, List<Integer> chunkIds, int attemptNumber) {

        Map<Integer, Future<ChunkResult>> futures = new LinkedHashMap<>();
        for (int i = 0; i < chunks.size(); i++) {
            int chunkId = chunkIds.get(i);
            ChunkSendTask task = new ChunkSendTask(chunkId, chunks.get(i), dwClient, attemptNumber);
            futures.put(chunkId, executor.submit(task));
        }

        Map<Integer, ChunkResult> results = new LinkedHashMap<>();
        futures.forEach((chunkId, future) -> {
            try {
                results.put(chunkId, future.get(2, TimeUnit.MINUTES));
            } catch (TimeoutException e) {
                future.cancel(true);
                results.put(chunkId, new ChunkResult(chunkId, chunks.get(chunkIds.indexOf(chunkId)),
                        false, 0, "Timed out", attemptNumber));
            } catch (ExecutionException | InterruptedException e) {
                results.put(chunkId, new ChunkResult(chunkId, chunks.get(chunkIds.indexOf(chunkId)),
                        false, 0, e.getMessage(), attemptNumber));
            }
        });
        return results;
    }

    private void sleepWithBackoff(int attempt) {
        try {
            long backoff = BASE_BACKOFF_MS * (1L << (attempt - 1)); // exponential: 1s, 2s, 4s...
            Thread.sleep(backoff);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    // ... chunkList, shutdownGracefully same as before
}
```

### Updated task to record attempt number
```java
class ChunkSendTask implements Callable<ChunkResult> {
    private final int chunkId;
    private final List<CardPayment> chunk;
    private final DWClient dwClient;
    private final int attemptNumber;

    ChunkSendTask(int chunkId, List<CardPayment> chunk, DWClient dwClient, int attemptNumber) {
        this.chunkId = chunkId;
        this.chunk = chunk;
        this.dwClient = dwClient;
        this.attemptNumber = attemptNumber;
    }

    @Override
    public ChunkResult call() {
        try {
            String payload = DWFormatter.toDWFormat(chunk);
            dwClient.send(payload, chunkId);
            return new ChunkResult(chunkId, chunk, true, chunk.size(), null, attemptNumber);
        } catch (Exception e) {
            return new ChunkResult(chunkId, chunk, false, chunk.size(), e.getMessage(), attemptNumber);
        }
    }
}
```

--- 
## Spring `@Async` + `TaskExecutor`
Spring's `@Async` abstracts away manual `ExecutorService` management — you configure a `TaskExecutor` bean once, and any method annotated `@Async` runs on it, returning a `CompletableFuture` instead of you manually calling `submit()`.

### Enable async + define the executor bean
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Bean(name = "dwSenderExecutor")
    public ThreadPoolTaskExecutor dwSenderExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("dw-sender-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    // Handles exceptions thrown from @Async void methods (not needed for Future-returning ones,
    // since those surface via the Future itself)
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
            System.err.printf("Async error in %s: %s%n", method.getName(), ex.getMessage());
    }
}
```

`ThreadPoolTaskExecutor` is Spring's wrapper around `java.util.concurrent.ThreadPoolExecutor` — same underlying engine, Spring-friendly configuration and lifecycle (starts/stops with the app context).

### The task as a Spring `@Service`, using `@Async`

```java
@Service
public class ChunkSenderService {

    private final DWClient dwClient;

    ChunkSenderService(DWClient dwClient) {
        this.dwClient = dwClient;
    }

    @Async("dwSenderExecutor")   // names which executor bean to run this on
    public CompletableFuture<ChunkResult> sendChunk(int chunkId, List<CardPayment> chunk, int attemptNumber) {
        try {
            String payload = DWFormatter.toDWFormat(chunk);
            dwClient.send(payload, chunkId);
            return CompletableFuture.completedFuture(
                new ChunkResult(chunkId, chunk, true, chunk.size(), null, attemptNumber));
        } catch (Exception e) {
            return CompletableFuture.completedFuture(
                new ChunkResult(chunkId, chunk, false, chunk.size(), e.getMessage(), attemptNumber));
        }
    }
}
```

Key Spring rule: `@Async` only works on public methods called from a different bean (Spring proxies the call — calling it from within the same class bypasses the proxy and runs synchronously).


### Orchestrator, now just composing `CompletableFuture`s
```java
@Service
public class CardPaymentEodOrchestrator {

    private static final int CHUNK_SIZE = 5_000;
    private static final int MAX_RETRIES = 3;

    private final CardPaymentRepository repository;
    private final ChunkSenderService chunkSenderService;

    CardPaymentEodOrchestrator(CardPaymentRepository repository, ChunkSenderService chunkSenderService) {
        this.repository = repository;
        this.chunkSenderService = chunkSenderService;
    }

    public EodReport runEodSend(LocalDate businessDate) {
        List<CardPayment> allPayments = repository.fetchCardPaymentsForDate(businessDate);
        List<List<CardPayment>> chunks = chunkList(allPayments, CHUNK_SIZE);

        Map<Integer, ChunkResult> results = submitAndAwait(chunks, IntStream.range(0, chunks.size()).boxed().toList(), 1);

        int attempt = 1;
        while (attempt < MAX_RETRIES) {
            var failed = results.entrySet().stream().filter(e -> !e.getValue().success()).toList();
            if (failed.isEmpty()) break;

            attempt++;
            var retryChunks = failed.stream().map(e -> e.getValue().chunkData()).toList();
            var retryIds = failed.stream().map(Map.Entry::getKey).toList();
            results.putAll(submitAndAwait(retryChunks, retryIds, attempt));
        }

        return buildReport(businessDate, new ArrayList<>(results.values()));
    }

    private Map<Integer, ChunkResult> submitAndAwait(List<List<CardPayment>> chunks, List<Integer> ids, int attempt) {
        Map<Integer, CompletableFuture<ChunkResult>> futures = new LinkedHashMap<>();
        for (int i = 0; i < chunks.size(); i++) {
            int chunkId = ids.get(i);
            futures.put(chunkId, chunkSenderService.sendChunk(chunkId, chunks.get(i), attempt));
        }

        // join() all futures — equivalent to Future.get() but unchecked
        CompletableFuture.allOf(futures.values().toArray(new CompletableFuture[0])).join();

        Map<Integer, ChunkResult> results = new LinkedHashMap<>();
        futures.forEach((id, future) -> results.put(id, future.join()));
        return results;
    }

    // chunkList, buildReport same as before
}
```

## What changed vs. raw `ThreadPoolExecutor`

| Raw approach | Spring `@Async` approach |
|---|---|
| `new ThreadPoolExecutor(...)` manually | `ThreadPoolTaskExecutor` bean, configured once, DI'd everywhere |
| `executor.submit(callable)` → `Future<T>` | `@Async` method call → `CompletableFuture<T>` automatically |
| Manual `shutdown()`/`awaitTermination()` | Spring manages executor lifecycle with the app context |
| `Future.get(timeout)` | `CompletableFuture.join()` (or `.get(timeout, unit)` if you want timeouts) |
| Task = `Callable` class you instantiate | Task = `@Async` method on a Spring-managed `@Service` |
| Exceptions surface at `future.get()` | Same for `CompletableFuture`, but void `@Async` methods route to `AsyncUncaughtExceptionHandler` |

## One important gotcha with `CompletableFuture.allOf`

`allOf(...).join()` waits for all futures but **doesn't propagate individual failures** — if one future completed exceptionally, `allOf().join()` itself throws, but you lose which one failed unless you handle it per-future. In our case we sidestep this because `sendChunk` never lets an exception escape — it always returns a *successful* `CompletableFuture` wrapping a `ChunkResult` that itself carries `success=false`. This is deliberate: it keeps failure a **data concern** (inspect `ChunkResult.success()`), not an **exception-handling concern** — cleaner when you're doing partial-failure reporting like this EOD job.

Want to also see this wired with **Spring's `@Retryable`** (from `spring-retry`) as a declarative alternative to the manual backoff loop?

