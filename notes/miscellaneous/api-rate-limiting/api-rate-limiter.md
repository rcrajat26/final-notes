Good — I'll cover all three keying strategies (per-account, per-IP, global) under both approaches: Resilience4j and a custom Guava-based filter. Skipping Bucket4j as requested.

## Setup

Using your `Client → Account` domain: `AccountController` exposes `GET /account/{accountId}`.

```java
@RestController
@RequestMapping("/account")
public class AccountController {

    private final AccountService accountService;

    public AccountController(AccountService accountService) {
        this.accountService = accountService;
    }

    @GetMapping("/{accountId}")
    public ResponseEntity<Account> getAccount(@PathVariable String accountId) {
        return ResponseEntity.ok(accountService.findById(accountId));
    }
}
```

Since it's a single server, everything below is **in-memory** — no Redis needed. If `AccountManagementService` ever scales horizontally, in-memory limiters become per-instance (so effective limit = configured limit × instance count), and you'd need a shared store (Bucket4j+Redis, or Resilience4j's `RateLimiterRegistry` doesn't natively support that — you'd swap algorithms entirely).

---

## Approach 1: Resilience4j

**Dependency** (Spring Boot 3 — use `resilience4j-spring-boot2` if you're on SB2):

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
```

### Global (simplest — one shared bucket, annotation-based)

```java
@GetMapping("/{accountId}")
@RateLimiter(name = "accountApi")
public ResponseEntity<Account> getAccount(@PathVariable String accountId) {
    return ResponseEntity.ok(accountService.findById(accountId));
}
```

```yaml
resilience4j:
  ratelimiter:
    instances:
      accountApi:
        limit-for-period: 50       # 50 requests
        limit-refresh-period: 1s   # per second
        timeout-duration: 0s       # don't block — fail fast
```

```java
@RestControllerAdvice
public class RateLimitExceptionHandler {

    @ExceptionHandler(RequestNotPermitted.class)
    public ResponseEntity<String> handleRateLimit(RequestNotPermitted ex) {
        return ResponseEntity.status(429).body("Rate limit exceeded");
    }
}
```

The annotation binds to **one named instance** — it has no notion of "per accountId" or "per IP" out of the box. For that you need a registry and a key resolver.

### Per-account / Per-IP (dynamic key via `RateLimiterRegistry`)

```java
@Component
public class KeyedRateLimiterRegistry {

    private final RateLimiterRegistry registry;
    private final RateLimiterConfig defaultConfig;

    public KeyedRateLimiterRegistry() {
        this.defaultConfig = RateLimiterConfig.custom()
                .limitForPeriod(20)
                .limitRefreshPeriod(Duration.ofSeconds(1))
                .timeoutDuration(Duration.ZERO)
                .build();
        this.registry = RateLimiterRegistry.of(defaultConfig);
    }

    public RateLimiter get(String key) {
        return registry.rateLimiter(key, defaultConfig); // creates on first use, reuses after
    }
}
```

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {

    private final KeyedRateLimiterRegistry rateLimiters;

    public RateLimitInterceptor(KeyedRateLimiterRegistry rateLimiters) {
        this.rateLimiters = rateLimiters;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws IOException {

        String key = resolveKey(request);
        RateLimiter limiter = rateLimiters.get(key);

        if (!limiter.acquirePermission()) {
            response.setStatus(429);
            response.setHeader("Retry-After", "1");
            response.getWriter().write("Rate limit exceeded for key: " + key);
            return false;
        }
        return true;
    }

    @SuppressWarnings("unchecked")
    private String resolveKey(HttpServletRequest request) {
        // --- Per-account ---
        Map<String, String> pathVars =
                (Map<String, String>) request.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE);
        return "account:" + pathVars.get("accountId");

        // --- Per-IP (swap in instead) ---
        // return "ip:" + request.getRemoteAddr();

        // --- Global (swap in instead) ---
        // return "global";
    }
}
```

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final RateLimitInterceptor rateLimitInterceptor;

    public WebConfig(RateLimitInterceptor rateLimitInterceptor) {
        this.rateLimitInterceptor = rateLimitInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(rateLimitInterceptor).addPathPatterns("/account/**");
    }
}
```

Note the `HandlerInterceptor.preHandle` runs **after** `DispatcherServlet` has resolved the handler mapping, so `HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE` already has `accountId` populated — no manual URI parsing needed. This won't be true for the Filter approach below.

---

## Approach 3: Custom filter (Guava `RateLimiter`)

**Dependencies:**

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>33.2.1-jre</version>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>3.1.8</version>
</dependency>
```

```java
@Component
public class SimpleRateLimitFilter extends OncePerRequestFilter {

    private static final AntPathMatcher PATH_MATCHER = new AntPathMatcher();
    private static final String PATTERN = "/account/{accountId}";
    private static final double PERMITS_PER_SECOND = 5.0;

    // Bounded + auto-evicting, so per-account/per-IP keys don't accumulate forever
    private final Cache<String, RateLimiter> limiters = Caffeine.newBuilder()
            .expireAfterAccess(Duration.ofMinutes(5))
            .maximumSize(10_000)
            .build();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {

        if (!PATH_MATCHER.match(PATTERN, request.getRequestURI())) {
            filterChain.doFilter(request, response);
            return;
        }

        String key = resolveKey(request);
        RateLimiter limiter = limiters.get(key, k -> RateLimiter.create(PERMITS_PER_SECOND));

        if (limiter.tryAcquire()) {
            filterChain.doFilter(request, response);
        } else {
            response.setStatus(429);
            response.getWriter().write("Rate limit exceeded");
        }
    }

    private String resolveKey(HttpServletRequest request) {
        // --- Per-account ---
        Map<String, String> vars = PATH_MATCHER.extractUriTemplateVariables(PATTERN, request.getRequestURI());
        return "account:" + vars.get("accountId");

        // --- Per-IP (swap in instead) ---
        // return "ip:" + request.getRemoteAddr();

        // --- Global (swap in instead) ---
        // return "global";
    }
}
```

Servlet filters run **before** `DispatcherServlet` maps the request to a handler, so `URI_TEMPLATE_VARIABLES_ATTRIBUTE` isn't populated yet — that's why this uses `AntPathMatcher.extractUriTemplateVariables` against the literal pattern instead of the request-attribute trick from the interceptor version. A naive `uri.substring(...)` would technically work here too but breaks the moment the path gains a suffix or query string.

`@Component`-annotated filters are auto-registered by Spring Boot; no `FilterRegistrationBean` needed unless you want to control ordering explicitly.

---

## A couple of things worth stress-testing

1. **Guava `RateLimiter` is a smoothed token bucket, not a concurrency limiter.** If you instead want "at most N requests in flight at once" (not "at most N per second"), that's a `Semaphore`, and it's a different problem — bursty-but-sparse traffic sails through a semaphore but not through a token bucket, and vice versa for sustained load with occasional spikes.

2. **Unbounded key growth is the real bug people ship.** With per-account or per-IP keying, both `RateLimiterRegistry.rateLimiter(key, config)` and a plain `ConcurrentHashMap<String, RateLimiter>` grow forever if you don't evict — that's why the filter above uses Caffeine with `expireAfterAccess`. The Resilience4j registry doesn't have built-in eviction, so on a real high-cardinality key (like per-IP with lots of unique clients) you'd want to either wrap it with a Caffeine cache the same way, or run a scheduled sweep removing stale entries.

3. **429 vs Retry-After:** Resilience4j's `RequestNotPermitted` gives you a clean exception to hang a `@RestControllerAdvice` on; the filter path has to write the response directly since it's below Spring MVC's exception-handling machinery.

---

**Trade-off summary:**

| | Resilience4j | Custom filter |
|---|---|---|
| Ecosystem | Micrometer metrics, circuit-breaker/retry composition for free | None — you own it |
| Dynamic per-key limiting | Needs registry + interceptor (annotation alone can't) | Native — key resolution is just a function |
| Algorithm | Fixed-window-ish permit refresh | Guava = smoothed token bucket (handles bursts differently) |
| Maintenance | Library upgrade risk | Your code, your bugs |

Given you're already using Resilience4j in the wallet platform for circuit breaking, the interceptor-based version slots in with tooling you already have wired up (Micrometer → your Prometheus/Grafana stack). The filter version is lighter but you're responsible for the eviction and algorithm correctness yourself.