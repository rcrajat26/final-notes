# CompletableFuture

- `CompletableFuture` is Java's tool for writing asynchronous, non-blocking code. 
- It's an evolution of the old Future interface — but where Future could only block-and-wait for a result, 
- CompletableFuture lets you chain, combine, and react to results without blocking.

## The problem 
```java
ExecutorService executor = Executors.newFixedThreadPool(2);
Future<Integer> future = executor.submit(() -> {
    Thread.sleep(1000);
    return 42;
});

int result = future.get(); // blocks the current thread until done
```

Issues:
- future.get() blocks — defeats the purpose of async code. 
- No way to chain "when this finishes, do that."
- No way to combine multiple futures easily. 
- No way to handle errors without try/catch around get().

## Solution: CompletableFuture
```java
// Runs asynchronously, returns a result
CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> {
    return computeSomething();
});

// Runs asynchronously, returns nothing (void)
CompletableFuture<Void> cf2 = CompletableFuture.runAsync(() -> {
    doSomething();
});

// Already-completed future (useful for testing / defaults)
CompletableFuture<String> done = CompletableFuture.completedFuture("hello");
```

By default, these run on the common ForkJoinPool. You can supply your own executor:
```java
ExecutorService myPool = Executors.newFixedThreadPool(4);
CompletableFuture.supplyAsync(() -> computeSomething(), myPool);
```

## Chaining: thenApply, thenAccept, thenRun
```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> 10)
    .thenApply(x -> x * 2)        // transform result: Integer -> Integer
    .thenApply(x -> x + 1);       // 10 -> 20 -> 21

future.thenAccept(x -> System.out.println("Result: " + x)); // consume, no return
future.thenRun(() -> System.out.println("Done!"));          // no access to result
``` 

| Method       | Input        | Output              |
|--------------|--------------|---------------------|
| `thenApply`  | Takes result | Returns a new value |
| `thenAccept` | Takes result | Returns nothing     |
| `thenRun`    | No input     | Returns nothing     |

### The Async variants
Every chaining method has an Async version:
```java
future.thenApply(x -> x * 2);          // runs on the same thread that completed the previous stage
future.thenApplyAsync(x -> x * 2);     // runs on ForkJoinPool.commonPool()
future.thenApplyAsync(x -> x * 2, myExecutor); // runs on your executor
```

**Rule of thumb**: use the Async variant with an explicit executor when the work is meaningful (I/O, CPU-heavy) so you're not surprised about which thread runs it.

## Chaining futures together: thenCompose
Use this when your callback itself returns a CompletableFuture (avoids nested futures):
```java
CompletableFuture<User> userFuture = getUserAsync(userId);

CompletableFuture<Order> orderFuture = userFuture.thenCompose(user -> 
    getOrdersForUserAsync(user)  // returns CompletableFuture<Order>
);
```

## Combining two independent futures: thenCombine
```java
CompletableFuture<Integer> priceFuture = getPriceAsync();
CompletableFuture<Double> taxRateFuture = getTaxRateAsync();

CompletableFuture<Double> total = priceFuture.thenCombine(taxRateFuture, 
    (price, taxRate) -> price * (1 + taxRate)
);
```
Both futures run concurrently, and the combiner runs once both complete.

## Waiting on multiple futures: allOf / anyOf
```java
CompletableFuture<Void> all = CompletableFuture.allOf(future1, future2, future3);
all.join(); // waits for all to complete, but discards individual results

// To collect results after allOf:
List<Integer> results = Stream.of(future1, future2, future3)
    .map(CompletableFuture::join)
    .collect(Collectors.toList());

// anyOf completes as soon as ANY one future completes
CompletableFuture<Object> any = CompletableFuture.anyOf(future1, future2, future3);
```

## Error handling 
```java
CompletableFuture.supplyAsync(() -> {
        if (Math.random() < 0.5) throw new RuntimeException("boom");
        return 42;
    })
    .exceptionally(ex -> {
        System.out.println("Recovered from: " + ex.getMessage());
        return -1; // fallback value
    })
    .thenAccept(result -> System.out.println("Final: " + result));
```

```java
// handle: runs regardless of success/failure, sees both
future.handle((result, ex) -> {
    if (ex != null) return -1;
    return result * 2;
});

// whenComplete: side-effect only, doesn't change the result, rethrows original error
future.whenComplete((result, ex) -> {
    if (ex != null) log.error("failed", ex);
    else log.info("succeeded: " + result);
});
```

| Method          | Can change result?  | Sees exception? | Sees success value? |
|-----------------|---------------------|-----------------|---------------------|
| `exceptionally` | Yes (only on error) | Yes             | No                  |
| `handle`        | Yes                 | Yes             | Yes                 |
| `whenComplete`  | No                  | Yes             | Yes                 |


## Getting the value out
```java
future.get();          // blocks, throws checked ExecutionException/InterruptedException
future.get(2, TimeUnit.SECONDS); // blocks with timeout
future.join();          // blocks, throws unchecked CompletionException instead — preferred in lambdas
```

## Putting it together — few realistic example
```java
CompletableFuture<String> pipeline = CompletableFuture
    .supplyAsync(() -> fetchUserId(), executor)
    .thenComposeAsync(userId -> fetchUserProfile(userId), executor)
    .thenApply(profile -> profile.getName().toUpperCase())
    .exceptionally(ex -> "UNKNOWN_USER")
    .thenApply(name -> "Hello, " + name);

pipeline.thenAccept(System.out::println);
```

### Java example
```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.concurrent.CompletableFuture;

public class ApiCallExample {

    private static final HttpClient client = HttpClient.newHttpClient();

    public static CompletableFuture<String> fetchUser(String userId) {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://api.example.com/users/" + userId))
                .GET()
                .build();

        return client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
                .thenApply(HttpResponse::body)          // extract response body
                .exceptionally(ex -> {
                    System.err.println("API call failed: " + ex.getMessage());
                    return "{}"; // fallback JSON
                });
    }

    public static void main(String[] args) {
        CompletableFuture<String> result = fetchUser("123");

        result.thenAccept(body -> System.out.println("Response: " + body));

        System.out.println("This prints immediately — call didn't block!");

        result.join(); // just so main() doesn't exit before the async call finishes
    }
}
```

### Chaining multiple API calls (real-world pattern)**
```java
public static CompletableFuture<String> fetchUserThenOrders(String userId) {
    return fetchUser(userId)                                // CF<String> (user JSON)
        .thenCompose(userJson -> {
            String parsedUserId = parseUserId(userJson);      
            return fetchOrders(parsedUserId);                 // returns CF<String> too
        })
        .exceptionally(ex -> "Could not fetch data: " + ex.getMessage());
}
```

### Spring Boot example
Typical pattern: `Controller` (API layer) → `Service` (business logic) → returns `CompletableFuture`. Spring MVC supports returning `CompletableFuture<T>` directly from a controller method — Spring automatically handles it asynchronously (frees up the servlet thread while waiting).

**Configure a dedicated executor (don't use the common pool for I/O!)**
```java
@Configuration
public class AsyncConfig {

    @Bean(name = "apiExecutor")
    public Executor apiExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("api-call-");
        executor.initialize();
        return executor;
    }
}
```

**Service class makes an API call**:
```java
@Service
public class UserService {

    private final HttpClient httpClient = HttpClient.newHttpClient();
    private final Executor apiExecutor;

    public UserService(@Qualifier("apiExecutor") Executor apiExecutor) {
        this.apiExecutor = apiExecutor;
    }

    public CompletableFuture<String> getUserById(String userId) {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://api.example.com/users/" + userId))
                .GET()
                .build();

        // java.net.http.HttpClient's sendAsync already runs off the calling thread,
        // but thenApplyAsync + apiExecutor ensures downstream processing 
        // also stays off Tomcat's request threads
        return httpClient.sendAsync(request, HttpResponse.BodyHandlers.ofString())
                .thenApplyAsync(HttpResponse::body, apiExecutor)
                .exceptionally(ex -> {
                    // log properly in real code
                    throw new RuntimeException("Failed to fetch user " + userId, ex);
                });
    }
}
```

**Controller class exposes this as an API**:
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public CompletableFuture<ResponseEntity<String>> getUser(@PathVariable String id) {
        return userService.getUserById(id)
                .thenApply(ResponseEntity::ok)
                .exceptionally(ex -> ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                        .body("Error: " + ex.getMessage()));
    }
}
```

**What happens under the hood**
- Request hits /api/users/123 → Spring's DispatcherServlet calls getUser("123").
- Since the return type is CompletableFuture<ResponseEntity<String>>, Spring releases the servlet thread immediately and registers a callback — it doesn't block a Tomcat thread waiting for the API call.
- When userService.getUserById() completes (on apiExecutor's threads), Spring resumes and sends the HTTP response.
- This means your servlet container can handle far more concurrent requests than blocking calls would allow, since threads aren't tied up waiting on slow external APIs.

## Miscellaneous 
### End-to-end flow: React frontend → Spring Boot (CompletableFuture) → back to screen

```
[React Component] 
      │  fetch('/api/users/123')
      ▼
[Browser sends HTTP GET request]
      │
      ▼
[Spring Boot: Tomcat thread picks up request]
      │  calls UserController.getUser("123")
      ▼
[Controller returns CompletableFuture<ResponseEntity<String>>]
      │  Tomcat thread is RELEASED immediately (doesn't block!)
      ▼
[Service: httpClient.sendAsync() fires request to external API]
      │  (runs on apiExecutor / HttpClient's internal threads)
      ▼
[External API responds after, say, 300ms]
      │
      ▼
[thenApplyAsync / thenApply chain runs → builds ResponseEntity]
      │
      ▼
[Spring resumes the original HTTP response, using a Tomcat thread again]
      │  Sends JSON back: { "name": "John Doe" }
      ▼
[Browser receives response]
      │
      ▼
[React: fetch promise resolves → setState → re-render]
      │
      ▼
[Screen shows "John Doe"]
```

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public CompletableFuture<ResponseEntity<UserDto>> getUser(@PathVariable String id) {
        return userService.getUserById(id)
                .thenApply(ResponseEntity::ok)
                .exceptionally(ex -> ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build());
    }
}

@Service
public class UserService {

    private final HttpClient httpClient = HttpClient.newHttpClient();
    private final ObjectMapper objectMapper = new ObjectMapper();
    private final Executor apiExecutor;

    public UserService(@Qualifier("apiExecutor") Executor apiExecutor) {
        this.apiExecutor = apiExecutor;
    }

    public CompletableFuture<UserDto> getUserById(String userId) {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://api.example.com/users/" + userId))
                .GET()
                .build();

        return httpClient.sendAsync(request, HttpResponse.BodyHandlers.ofString())
                .thenApplyAsync(response -> {
                    try {
                        return objectMapper.readValue(response.body(), UserDto.class);
                    } catch (JsonProcessingException e) {
                        throw new RuntimeException("Failed to parse user JSON", e);
                    }
                }, apiExecutor);
    }
}
```

| Time             | Thread                                       | What's happening                                                                                                           |
|------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| **t = 0 ms**     | Tomcat thread (e.g. `http-nio-8080-exec-1`)  | Receives request, calls controller, gets `CompletableFuture` back, **returns immediately and is freed** for other requests |
| **t = 0–300 ms** | `HttpClient` internal thread / `apiExecutor` | Calls external API, waits for response, and parses JSON                                                                    |
| **t = 300 ms**   | `apiExecutor` thread                         | `thenApply(ResponseEntity::ok)` builds the response object                                                                 |
| **t = 300 ms**   | A Tomcat thread (possibly a different one)   | Spring's async support writes the actual HTTP response bytes back to the client                                            |


- The browser sends GET /api/users/123, and Frontend is waiting on a Promise. 
- This is the entire point of using CompletableFuture here: during those 300ms of waiting on the external API, the Tomcat thread that handled the original request is free to serve other incoming requests — it's not sitting idle blocked on I/O. 
- If you had 200 concurrent users each waiting on a slow API, a blocking approach would need ~200 Tomcat threads tied up; the async approach needs far fewer.

**Know these**:
- **Does the frontend get a special response/status code when Tomcat releases the thread**
  - No. This is a common misconception — there's no intermediate response, no special status code (like a 202 Accepted placeholder), nothing sent early.
  - From the client's perspective, this is a completely ordinary HTTP request. The TCP connection stays open, the browser just waits, exactly as it would for any slow endpoint
  - Only one HTTP response is ever sent — at t=300ms, when the actual data is ready — with a normal status code like 200 OK.
  - "Releasing the Tomcat thread" is a purely server-internal implementation detail.
- **Here's what actually happens under the hood, when tomcat releases the thread**
  - The client's socket connection to the server stays open the whole time
  - Spring uses async request processing — when your controller returns a CompletableFuture, Spring puts the servlet request into "async mode" (AsyncContext), detaches it from the current worker thread, and stores a reference to it.
  - When the CompletableFuture completes, Spring's async support wakes up the request, reattaches it to a worker thread (which may be a different one), and continues processing to send the final response.
  

### Scenario: 30 requests arrive within a 300ms window, each needing a 300ms external API call
**Without CompletableFuture (blocking, e.g. RestTemplate)**
- Tomcat has a fixed thread pool — default is 200, but let's use a small number to make the bottleneck obvious, say 10 threads

| Time           | What's happening                                                                                                                                |
|----------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| **t = 0 ms**   | 30 requests arrive. 10 grab a Tomcat thread each and start blocking on the external call. **20 requests sit in Tomcat's queue, doing nothing.** |
| **t = 300 ms** | First 10 finish, threads are freed, next 10 are picked up and start their blocking calls.                                                       |
| **t = 600 ms** | Next 10 finish, last 10 are picked up.                                                                                                          |
| **t = 900 ms** | Last batch finishes.                                                                                                                            |

- Total time to serve all 30: ~900ms — because your Tomcat thread pool itself was the bottleneck, and requests literally queued up waiting for a thread to become free. 
- This is a real, well-known problem in blocking servers — it's called thread starvation, and it does cause a backlog.

**With CompletableFuture (non-blocking, async)**
- Now Tomcat threads are released instantly after handoff. 
- All 30 requests get accepted and dispatched almost immediately — Tomcat's thread pool is no longer the bottleneck.

| Time           | What's happening                                                                                                                                                     |
|----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **t = 0 ms**   | All 30 requests arrive and are immediately accepted. Tomcat threads are freed right after handing off the work. All 30 external API calls fire off **concurrently**. |
| **t = 300 ms** | All 30 external calls return around the same time. Responses are written back to the clients.                                                                        |
- Total time to serve all 30: ~300ms — roughly the same as serving one request. 
- This is the actual benefit: removing the artificial thread-pool ceiling lets throughput scale with the downstream API's real concurrency limits, not your web server's internal thread count.


**The bottleneck doesn't disappear — it moves, to two places:**
- Your own executor / HTTP client's connection pool. 
  - If apiExecutor only has 10 threads, or HttpClient's internal connection pool caps concurrent connections at some number, you get the same queuing behavior as the blocking scenario above — just shifted one layer down. 
  - This is tunable and often desirable (see below)
- The external API's actual capacity. 
  - If you blast 30 (or 3,000) truly concurrent calls at a third-party API, that system might not have unlimited capacity — you could overwhelm their servers, hit their rate limits, or get throttled/429'd. 
  - This is the real "stampede" risk — and it's completely valid.

> Important nuance: this stampede risk exists regardless of blocking vs. async

**The correct fix: intentional concurrency control, not relying on thread starvation**
- Instead of accidentally throttling via a small Tomcat pool, you deliberately control concurrency at the right layer:
  - Bound your apiExecutor to a sensible size based on what the external API can actually handle (e.g., if they support 50 concurrent connections comfortably, size around there).
  - Use a Semaphore to cap in-flight calls to the external API explicitly:
    ```java
    private final Semaphore rateLimiter = new Semaphore(50); // max 50 concurrent calls
    
    public CompletableFuture<UserDto> getUserById(String userId) {
        try {
            rateLimiter.acquire();
            return httpClient.sendAsync(request, HttpResponse.BodyHandlers.ofString())
                    .thenApply(this::parse)
                    .whenComplete((r, ex) -> rateLimiter.release());
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
    ```
  - Add a circuit breaker (e.g., `Resilience4j`) so if the external API starts failing/slowing under load, you stop hammering it and fail fast instead. 
  - Add a queue with backpressure (reject with 429 Too Many Requests past a threshold) rather than accepting unlimited work.

