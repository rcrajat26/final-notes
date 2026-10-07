# 07 — Spring MVC

## M24. Spring MVC Foundations

### T1. A request three ways — `MK` `[P→S→B]`
- `GET /clients/{id}` in a plain `HttpServlet` → `DispatcherServlet` + `@Controller` → Boot's `WebMvcAutoConfiguration`.

### T2. Servlet recap — `MK`
- The container, thread-per-request, filters, the servlet context.
- `[3.x]` `javax.servlet` → `jakarta.servlet`.

### T3. The request lifecycle — `MK`
- In order:
  1. Filter chain.
  2. `DispatcherServlet` → `HandlerMapping` → interceptors.
  3. `HandlerAdapter` → argument resolvers → your method.
  4. Return-value handlers → message converters.
  5. `afterCompletion`.

### T4. Special bean types — `MK`
- `HandlerMapping`, `HandlerAdapter`, `HandlerExceptionResolver`, `ViewResolver`, `LocaleResolver`, `MultipartResolver`.

### T5. Context hierarchy — `GTH`
- Root vs servlet application context in classic MVC; Boot uses a single context.

### T6. Path matching — `MK`
- `AntPathMatcher` vs `PathPatternParser` (Boot's default since 2.6).
- `[3.x]` Trailing-slash matching is removed in Framework 6.

### T7. Server-side views — `GTH`
- A brief look at Thymeleaf.

### Traps
- `/clients/` stops matching `/clients` after upgrading to 3.x.

---

## M25. Building REST APIs

### T1. Controllers and resource design — `MK`
- `@RestController` and the composed mappings (`@GetMapping` etc.).
- Resource design: `/clients`, `/clients/{id}/accounts`, `/accounts/{id}/status`.

### T2. Request binding — `MK`
- `@PathVariable`, `@RequestParam`, `@RequestHeader`, `@RequestBody`, `@ModelAttribute`, `@CookieValue`.
- Binding enums such as `AccountType`.
- `GTH` `@MatrixVariable`.

### T3. Responses — `MK`
- `ResponseEntity`, status codes, headers.
- `Location` headers via `UriComponentsBuilder`; 201 vs 204.

### T4. Message conversion — `MK`
- `HttpMessageConverter`.
- Customizing Jackson (`spring.jackson.*`, builder customizers); `java.time` handling.
- Record DTOs vs domain objects.
- `GTH` `@JsonView`.

### T5. Content negotiation — `MK`
- `produces` / `consumes`, the `Accept` header, 406 and 415 responses.

### T6. Pagination and filtering — `GTH`
- Query-parameter conventions, and a pointer to `Pageable`.

### T7. API versioning — `MK`
- URI, header and media-type approaches, built by hand.
- `[4.x]` Built-in API versioning.

### T8. Async request handling — `GTH`
- `Callable`, `DeferredResult`, `CompletableFuture`, `StreamingResponseBody`, `SseEmitter`.

### T9. HATEOAS and API docs — `GTH`
- Spring HATEOAS; springdoc-openapi (1.x for 2.7, 2.x for `[3.x]`).

### T10. CORS — `MK`
- `@CrossOrigin`, global configuration, and how CORS interacts with Spring Security.

### Traps
- Returning entities instead of DTOs.
- Jackson's unknown-property defaults.
- Enum binding being case-sensitive.

---

## M26. Error Handling and Request Validation

### T1. Boot's default error handling — `MK`
- `BasicErrorController`, the `/error` path, `ErrorAttributes`.
- `server.error.*` properties (`include-message`, `include-stacktrace`).

### T2. @ExceptionHandler and @ControllerAdvice — `MK`
- Controller-local vs global handlers; how a handler is matched; scoping and ordering advice.
- `ResponseEntityExceptionHandler` as a base class.

### T3. The exception resolver chain — `MK`
- `ExceptionHandlerExceptionResolver`, `ResponseStatusExceptionResolver`, `DefaultHandlerExceptionResolver`.
- `@ResponseStatus` and `ResponseStatusException`.

### T4. Error payload design — `MK`
- A custom error DTO in 2.7.
- `[3.x]` `ProblemDetail` / `ErrorResponse` (RFC 7807), enabled with `spring.mvc.problemdetails.enabled`.

### T5. Validation at the edge — `MK`
- `@Valid @RequestBody` → `MethodArgumentNotValidException` → 400.
- `@Validated` on a controller for parameters → `ConstraintViolationException`.
- `BindingResult`.
- `[3.x]` `HandlerMethodValidationException`.

### T6. Localized errors — `GTH`
- See M17.

### T7. Errors from filters and security — `MK`
- Why errors thrown in filters (including Spring Security) bypass `@ControllerAdvice`.

### Traps
- Leaking stack traces in error responses.
- Two advice classes that both match the same exception.

---

## M27. Web Extension Points and the Embedded Server

### T1. Filters — `MK`
- `Filter` vs `OncePerRequestFilter`.
- `FilterRegistrationBean` for ordering and URL patterns.
- Example: a request-logging filter that sets up the MDC.

### T2. HandlerInterceptor — `MK`
- `preHandle`, `postHandle`, `afterCompletion`.

### T3. Filter vs interceptor vs aspect — `MK`
- A comparison table, and how the three nest.

### T4. WebMvcConfigurer — `MK` (condensed)
- Interceptors, CORS, formatters, message converters.
- A custom argument resolver for `@CurrentClient`.

### T5. The @EnableWebMvc trap — `MK`

### T6. Embedded Tomcat configuration — `MK`
- `server.port`, `server.servlet.context-path`.
- Capacity defaults: 200 max threads, accept-count 100, 8192 max connections.
- Timeouts, compression, HTTP/2; `WebServerFactoryCustomizer`.
- `GTH` SSL.

### T7. Multipart uploads — `GTH`

### Traps
- A filter executing twice on forward/async dispatches.
- A filter registered twice (once as a `@Component`, once via a registration bean).
