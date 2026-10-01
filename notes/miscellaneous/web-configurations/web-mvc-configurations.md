`WebMvcConfigurer` is Spring Boot's escape hatch for customizing Spring MVC without dropping the auto-configuration Boot already gives you (message converters, static resource handling, etc.). It's a set of default (no-op) methods you override selectively — you don't need `@EnableWebMvc` for this, and in fact you generally shouldn't (more on that at the end).

Here's every commonly-used hook, in the order you're most likely to reach for them.

---

## 1. Interceptors — `addInterceptors`

Already covered for rate limiting. Worth adding: **ordering and exclusions** when you have more than one.

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(authInterceptor)
            .addPathPatterns("/account/**")
            .order(1);

    registry.addInterceptor(rateLimitInterceptor)
            .addPathPatterns("/account/**")
            .excludePathPatterns("/account/health")
            .order(2);
}
```

Interceptors run in registration order for `preHandle`, and **reverse order** for `postHandle`/`afterCompletion` — same as a stack. So `authInterceptor` should almost always be `order(1)` (reject unauthenticated requests before you spend a rate-limit permit on them), not the other way round.

---

## 2. CORS — `addCorsMappings`

If `AccountManagementService` is ever called from a browser-based frontend on a different origin:

```java
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/account/**")
            .allowedOrigins("https://internal-dashboard.ig.com")
            .allowedMethods("GET", "POST")
            .allowedHeaders("Authorization", "Content-Type")
            .allowCredentials(true)
            .maxAge(3600);
}
```

Global config here beats scattering `@CrossOrigin` across controllers. Note: `allowedOrigins("*")` and `allowCredentials(true)` are **mutually exclusive** — Spring throws at startup if you combine them; use `allowedOriginPatterns` if you need wildcard + credentials.

---

## 3. Static resources — `addResourceHandlers`

Rare for a pure REST backend, but shows up if `AccountManagementService` also serves something like API docs or a health dashboard:

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/docs/**")
            .addResourceLocations("classpath:/static/docs/")
            .setCacheControl(CacheControl.maxAge(1, TimeUnit.DAYS));
}
```

If you don't touch this at all, Boot's defaults (`classpath:/static/`, `/public/`, etc.) still work — this method only matters when you need extra locations or cache headers.

---

## 4. Simple view/redirect mappings — `addViewControllers`

For routes that don't need a controller method at all — no business logic, just a redirect or a trivial view:

```java
@Override
public void addViewControllers(ViewControllerRegistry registry) {
    registry.addRedirectView("/", "/actuator/health");
}
```

Avoids writing a whole `@Controller` class for a one-line redirect. Not relevant if everything's a JSON API, but common in services that expose a root health/landing route.

---

## 5. Path matching — `configurePathMatch`

Controls how request URLs get matched to `@RequestMapping` patterns:

```java
@Override
public void configurePathMatch(PathMatchConfigurer configurer) {
    configurer.setUseTrailingSlashMatch(false); // /account/123/ != /account/123 (Boot 3 default anyway)
}
```

In Spring Boot 3 / Spring Framework 6, trailing-slash matching is **off by default** (a deliberate security-hardening change — request-smuggling concerns), so `/account/123/` returns 404 unless you explicitly re-enable it. If you're migrating from Boot 2, this is a common "why did my API break" surprise.

---

## 6. Content negotiation — `configureContentNegotiation`

Determines how Spring picks a response format when a client's `Accept` header is ambiguous or absent:

```java
@Override
public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
    configurer.defaultContentType(MediaType.APPLICATION_JSON)
              .favorParameter(false)  // don't let ?format=xml override the Accept header
              .ignoreAcceptHeader(false);
}
```

For a pure JSON REST service you often don't need to touch this — Boot defaults to JSON already — but it matters the moment you support both JSON and XML, or want to lock a client-facing API down to JSON-only regardless of what a caller's `Accept` header requests.

---

## 7. Formatters and converters — `addFormatters`

This is the one you'll actually hit often. It handles **String → Type** conversion for `@PathVariable` / `@RequestParam` (not request *bodies* — that's Jackson's job, see #8).

Example: your `Account` domain has an `accountType` (`IG`, `CFT`, `IGSTK`). If a query param needs to bind that:

```java
@GetMapping("/search")
public List<Account> search(@RequestParam AccountType accountType) { ... }
```

Plain enums bind automatically via Spring's built-in `StringToEnumConverterFactory` (case-sensitive, exact name match). But if you want case-insensitive matching, or the client sends a different string than the enum constant name, register a custom converter:

```java
@Override
public void addFormatters(FormatterRegistry registry) {
    registry.addConverter(new Converter<String, AccountType>() {
        @Override
        public AccountType convert(String source) {
            return AccountType.valueOf(source.trim().toUpperCase());
        }
    });
}
```

Another very common one: a custom ID type wrapper (if `accountId` is a value object rather than a raw `String`/`Long`):

```java
registry.addConverter(new Converter<String, AccountId>() {
    @Override
    public AccountId convert(String source) {
        return AccountId.of(source);
    }
});
```

Then your controller can accept the domain type directly:

```java
@GetMapping("/{accountId}")
public ResponseEntity<Account> getAccount(@PathVariable AccountId accountId) { ... }
```

A malformed value throws inside the converter — pair this with a `@ExceptionHandler(ConversionFailedException.class)` (or let it surface as Spring's default 400) rather than validating the raw string separately.

---

## 8. Message converters — `configureMessageConverters` / `extendMessageConverters`

This is for **request/response body** serialization (JSON, in most cases) — different from #7.

```java
@Override
public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
    converters.stream()
            .filter(MappingJackson2HttpMessageConverter.class::isInstance)
            .map(MappingJackson2HttpMessageConverter.class::cast)
            .findFirst()
            .ifPresent(c -> c.getObjectMapper().registerModule(new JavaTimeModule()));
}
```

Use `extendMessageConverters` (adds to Boot's existing list — Boot's auto-configured `ObjectMapper` bean stays intact) rather than `configureMessageConverters` (replaces the list entirely — you lose Boot's defaults unless you re-add everything by hand). The former is almost always what you want; the latter is a common footgun when people copy old Spring-XML-era examples.

If you just need to tweak Jackson behavior (naming strategy, date format, null handling), it's usually cleaner to define an `ObjectMapper` `@Bean` or set `spring.jackson.*` properties than to reach for this method at all — reserve `extendMessageConverters` for adding a converter for a *non-JSON* format (e.g., protobuf, CSV export endpoint).

---

## 9. Custom argument resolvers — `addArgumentResolvers`

Lets you inject a custom object into a controller method parameter, resolved from the request. Classic use: extracting an authenticated caller's identity from a header/JWT without repeating that logic in every controller.

```java
@Override
public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    resolvers.add(new AuthenticatedClientArgumentResolver(jwtService));
}
```

```java
public class AuthenticatedClientArgumentResolver implements HandlerMethodArgumentResolver {

    private final JwtService jwtService;

    public AuthenticatedClientArgumentResolver(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterType().equals(AuthenticatedClient.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                   NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
        String token = webRequest.getHeader("Authorization");
        return jwtService.parseClient(token); // throws -> 401 via @ExceptionHandler
    }
}
```

```java
@GetMapping("/{accountId}")
public ResponseEntity<Account> getAccount(@PathVariable AccountId accountId,
                                           AuthenticatedClient caller) {
    accountService.assertOwnership(caller, accountId);
    return ResponseEntity.ok(accountService.findById(accountId));
}
```

This is the idiomatic Spring MVC alternative to reading `HttpServletRequest` manually in every method — it centralizes the "how do I get the caller" logic in one place instead of the interceptor-then-`request.getAttribute()` dance.

---

## 10. Async support — `configureAsyncSupport`

Relevant if any controller method returns `Callable<T>`, `DeferredResult<T>`, or you're on virtual threads and want async servlet dispatch tuned explicitly:

```java
@Override
public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
    configurer.setDefaultTimeout(5000); // ms, before an async request times out with 503
    configurer.setTaskExecutor(myAsyncTaskExecutor);
}
```

With Java 21 virtual threads and `spring.threads.virtual.enabled=true`, most people don't need this anymore — blocking calls on virtual threads don't pin the platform thread pool the way they used to, so the classic reason to reach for `Callable`/`DeferredResult` (freeing up a scarce Tomcat worker thread during a slow downstream call) mostly goes away. Still relevant if you have genuinely async, non-blocking work (e.g., a `CompletableFuture` from a reactive downstream client).

---

## 11. Custom validator — `getValidator`

Overrides Spring's default `Validator` used by `@Valid`/`@Validated` — rare, since Boot already wires up a Bean Validation (`jakarta.validation`) provider automatically. Only override this if you need a validator that *isn't* JSR-380-based, or want to compose multiple validators:

```java
@Override
public Validator getValidator() {
    return new CompositeValidator(List.of(new AccountOwnershipValidator(), defaultBeanValidator));
}
```

Most people never touch this — annotation-based `@NotNull`/`@Size`/custom `@Constraint` on the DTO is the standard path and doesn't require overriding anything here.

---

## Ordering multiple `WebMvcConfigurer` beans

Spring **merges** all `WebMvcConfigurer` beans in the context rather than requiring exactly one — you can split by concern (`RateLimitWebConfig`, `CorsWebConfig`, `ConverterWebConfig`) instead of one giant class. If two beans register interceptors, both registrations apply; if ordering across *classes* matters, use `@Order`:

```java
@Configuration
@Order(1)
public class SecurityWebConfig implements WebMvcConfigurer { ... }

@Configuration
@Order(2)
public class RateLimitWebConfig implements WebMvcConfigurer { ... }
```

---

## The pitfall to know about: `@EnableWebMvc`

Adding `@EnableWebMvc` on top of a `WebMvcConfigurer` switches Spring Boot from its **auto-configured** MVC setup to **manual** mode — you inherit `WebMvcConfigurationSupport`'s bare defaults and lose Boot's opinionated auto-configuration (Jackson auto-config for message converters, static resource handling defaults, error page handling, etc.) unless you rebuild it yourself. `WebMvcConfigurer` alone is a set of *callbacks into* Boot's existing auto-configuration — that's the whole point of the interface, and 95% of Spring Boot apps should never need `@EnableWebMvc` at all. If you ever see a tutorial pairing the two, that's usually outdated pre-Boot-era Spring guidance leaking in.