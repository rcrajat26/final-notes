# 10 — Security

## M34. Spring Security in Boot
(The filter-chain parts are condensed, since that ground is already covered.)

### T1. Security three ways — `MK` `[P→S→B]`
- An auth check in a hand-written servlet filter → the Spring Security filter chain → Boot's defaults (generated password, every endpoint secured).

### T2. Architecture recap — `MK`
- `DelegatingFilterProxy` → `FilterChainProxy` → one or more `SecurityFilterChain`s.
- The key filters and their order.
- `SecurityContextHolder` and its strategies.
- `Authentication`, `AuthenticationManager`, `ProviderManager`, `AuthenticationProvider`.

### T3. Configuration styles — `MK`
- In 2.7: `WebSecurityConfigurerAdapter` (deprecated in 5.7) → a `SecurityFilterChain` bean (the recommended style).
- `[3.x]`:
  - `antMatchers` / `mvcMatchers` → `requestMatchers`.
  - `authorizeRequests` → `authorizeHttpRequests`.
- `[4.x]` Security 7 is lambda-DSL only.

### T4. Authentication — `MK`
- `UserDetailsService`; `PasswordEncoder` (delegating, bcrypt).
- Basic and form login; stateless APIs.

### T5. Authorization — `MK`
- URL rules for the client and account endpoints; roles vs authorities.
- `[3.x]` `AuthorizationManager`.

### T6. CSRF, CORS, headers and sessions — `MK`
- CSRF decisions for stateless REST; CORS integration; security headers; session management.

### T7. Exception handling — `MK`
- `AuthenticationEntryPoint` and `AccessDeniedHandler`; 401 vs 403.
- Why security errors never reach `@ControllerAdvice`.

### T8. Multiple filter chains — `MK`
- Separate actuator and API chains; `@Order`; `securityMatcher`.

### Traps
- Disabling CSRF without understanding why.
- Rule ordering where `anyRequest()` comes first.
- A missing `securityMatcher` that shadows another chain.

---

## M35. OAuth2, JWT and Method Security

### T1. OAuth2 and OIDC primer — `MK`
- Roles; the client-credentials and authorization-code + PKCE flows; JWT anatomy; JWKS.

### T2. Resource server — `MK`
- `spring-boot-starter-oauth2-resource-server`; `issuer-uri` / `jwk-set-uri`.
- `JwtAuthenticationConverter`: mapping scopes to authorities.
- `GTH` Opaque tokens.

### T3. OAuth2 client — `MK`
- Client credentials for `client-service` → `account-service`.
- The authorized-client manager wired into `WebClient` or a `RestTemplate` interceptor.
- `[3.x]` `RestClient` support.

### T4. Method security — `MK`
- `@EnableGlobalMethodSecurity` → `@EnableMethodSecurity` (available since 5.6; the required style in `[3.x]`).
- `@PreAuthorize` ownership checks (does this client own this account?) using SpEL and bean references.
- `@PostAuthorize`; `GTH` `@PostFilter`.
- Proxy limits apply.

### T5. Propagating the security context — `MK`
- To `@Async` (delegating executors), into `WebClient` calls, and in reactive code.

### T6. Spring Authorization Server — `GTH`

### T7. Testing security — `MK`
- `@WithMockUser`; `jwt()` and `csrf()` request post-processors; a custom `@WithSecurityContext`.

### Traps
- Role prefix confusion (`ROLE_`).
- `@PreAuthorize` on a `private` method.
- Trusting JWTs without validating issuer and audience.
