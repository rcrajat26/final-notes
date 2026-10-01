Separate — and it's worth being precise about *why*, because it changes where things like your rate limiter should live.

## Different layer, different mechanism

`WebMvcConfigurer` configures things **inside** `DispatcherServlet` — interceptors, argument resolvers, converters, all the hooks from the previous answer. Spring Security instead registers itself as a single servlet `Filter` (`DelegatingFilterProxy` → `FilterChainProxy`) that sits **in front of** `DispatcherServlet` entirely, at the servlet container level.

Request pipeline, in order:

```
Servlet Container
  → Spring Security's FilterChainProxy (a chain of filters: CORS, CSRF, authentication, authorization...)
    → DispatcherServlet
      → HandlerInterceptors (your rate-limit interceptor, auth interceptor)
        → Controller method (with argument resolvers, converters applied)
```

This matters concretely: **by the time your `HandlerInterceptor` from the earlier answer runs, Spring Security has already authenticated and authorized the request.** A rejected/unauthenticated request never reaches your interceptor at all — it's turned away in the filter chain, before `DispatcherServlet` is even invoked. So if your threat model includes rate-limiting *unauthenticated* traffic (e.g., someone hammering a login endpoint or probing with garbage JWTs), an MVC interceptor is too late in the pipeline; you need a `Filter` positioned inside the security chain instead. More on that below.

---

## Modern configuration: `SecurityFilterChain` bean

If you last touched Spring Security via `WebSecurityConfigurerAdapter`, that's gone — deprecated in Security 5.7, removed entirely in Security 6 / Spring Boot 3. Current style is a `SecurityFilterChain` `@Bean`, and it works the same way in Boot 2 (Security 5.7+) and Boot 3 (Security 6):

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable()) // stateless JSON API — no session, no CSRF token to protect
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/docs/**").permitAll()
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll() // CORS preflight — see gotcha below
                .requestMatchers("/account/**").authenticated()
                .anyRequest().denyAll()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults())); // if callers present JWTs

        return http.build();
    }
}
```

`.csrf().disable()` is correct here specifically *because* it's a stateless JSON API with token-based auth, not a browser session — CSRF protection exists to defend session-cookie-based auth, and disabling it on a stateless resource server is standard, not a shortcut. If `AccountManagementService` ever serves a session-authenticated browser client, that changes.

---

## Method-level authorization for `AccountController`

Given your `Client → Account` ownership model, endpoint-level `authorizeHttpRequests` usually isn't fine-grained enough — you need "is this caller allowed to see *this* `accountId`," not just "is this caller authenticated." That's `@PreAuthorize` with a custom expression, or an explicit check in the service:

```java
@Configuration
@EnableMethodSecurity // enables @PreAuthorize / @PostAuthorize
public class MethodSecurityConfig {}
```

```java
@GetMapping("/{accountId}")
@PreAuthorize("@accountAuthz.canAccess(authentication, #accountId)")
public ResponseEntity<Account> getAccount(@PathVariable AccountId accountId) {
    return ResponseEntity.ok(accountService.findById(accountId));
}
```

```java
@Component("accountAuthz")
public class AccountAuthorizationService {
    public boolean canAccess(Authentication auth, AccountId accountId) {
        String clientId = auth.getName(); // from JWT subject, or however you populate it
        return accountService.findById(accountId).getClientId().equals(clientId);
    }
}
```

This is arguably cleaner than the `AuthenticatedClient` custom argument resolver from the previous answer for authorization specifically — `@PreAuthorize` runs as a method interceptor (AOP) *before* the controller method body even starts, so an unauthorized caller never touches `accountService.findById`. The argument resolver is still the right tool if you just need the caller's identity injected as a parameter for other purposes.

---

## Where CORS actually needs to live

This is the most common integration gotcha. `WebMvcConfigurer.addCorsMappings` alone **stops working correctly once Spring Security is on the classpath**, because CORS preflight (`OPTIONS`) requests hit Security's filter chain first — and by default Security has no idea about your CORS mapping, so it can reject the preflight before MVC's CORS logic ever runs.

Fix: define a single `CorsConfigurationSource` bean and wire it into Security explicitly. Spring Security's `.cors()` will delegate to a `CorsConfigurationSource` bean if one exists — you don't need to duplicate the config in both places.

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://internal-dashboard.ig.com"));
    config.setAllowedMethods(List.of("GET", "POST"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/account/**", config);
    return source;
}
```

```java
http.cors(Customizer.withDefaults()) // picks up the CorsConfigurationSource bean above
```

At that point, drop `addCorsMappings` from your `WebMvcConfigurer` entirely — having both is redundant at best, and a source of "which config actually applies" confusion at worst.

---

## Exception handling: filter-level vs MVC-level

Your `@RestControllerAdvice` from the rate-limiting answer only catches exceptions thrown **inside** `DispatcherServlet`'s handling — i.e., inside controllers, interceptors, argument resolvers. An authentication failure or access-denial happens *inside Security's filter*, before `DispatcherServlet` runs, so `@ExceptionHandler` never sees it. Security has its own equivalent hooks:

```java
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint((request, response, authException) -> {
        response.setStatus(401);
        response.getWriter().write("Unauthenticated");
    })
    .accessDeniedHandler((request, response, accessDeniedException) -> {
        response.setStatus(403);
        response.getWriter().write("Forbidden");
    })
);
```

---

## Slotting your rate limiter into the security filter chain

This is the piece that actually answers the "same or separate" question with a concrete decision. You have two real options, and they answer different threat models:

**Option A — keep it as an MVC interceptor** (as built earlier). Rate-limits only requests that already passed authentication. Fine if your concern is "an authenticated caller hammering the endpoint," not abuse of the auth mechanism itself.

**Option B — register it as a `Filter` inside the security chain**, positioned *before* authentication, so unauthenticated/malformed requests get rate-limited before Spring Security spends any effort validating them (JWT signature checks aren't free, and they're a classic vector for cheap DoS if unthrottled):

```java
http.addFilterBefore(simpleRateLimitFilter, UsernamePasswordAuthenticationFilter.class);
```

Or, if you're on the resource-server/JWT setup above and want it before JWT validation specifically:

```java
http.addFilterBefore(simpleRateLimitFilter, BearerTokenAuthenticationFilter.class);
```

Your `SimpleRateLimitFilter` from the earlier answer (`OncePerRequestFilter`) drops into either position unchanged — no rewrite needed, just a different registration point. One adjustment worth making: for pre-authentication placement, key on IP rather than `accountId`/client identity, since you don't have a verified identity yet at that point in the chain — that's exactly the "per-IP" variant already sketched in the filter's `resolveKey`.

---

## Quick mental model to keep

| | `WebMvcConfigurer` | Spring Security |
|---|---|---|
| Extension point | Callback interface, hooks into Boot's MVC auto-config | `SecurityFilterChain` bean, a chain of servlet `Filter`s |
| Runs | Inside `DispatcherServlet` | Before `DispatcherServlet`, at the filter level |
| Governs | Interceptors, converters, CORS *mapping*, argument resolvers | Authentication, authorization, CSRF, CORS *enforcement* |
| Exceptions caught by | `@RestControllerAdvice` | `AuthenticationEntryPoint` / `AccessDeniedHandler` |

They're configured independently, but CORS and rate-limiting are the two places they genuinely need to agree with each other rather than each silently assuming it owns the whole picture.