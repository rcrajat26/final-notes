# 03 — Aspect-Oriented Programming

## M10. The Proxy Model

### T1. Plain Java proxies — `MK` `[P→S→B]`
- A hand-written decorator around `AccountService`, then the same with `java.lang.reflect.Proxy` and an `InvocationHandler`.
- Example concern: audit logging of account status changes.

### T2. Subclass proxies — `MK`
- The CGLIB concept (Spring ships its own repackaged copy) and Objenesis.
- Does the constructor run? Field values on the proxy vs on the target.
- `GTH` Byte Buddy, for comparison.

### T3. How Spring picks a proxy type — `MK`
- The selection rule in `DefaultAopProxyFactory`.
- Boot defaults `spring.aop.proxy-target-class=true` (since 2.0), so injecting by concrete class always works.

### T4. What cannot be advised — `MK`
- `final` classes and methods, `private` and `static` methods, records (which are `final`), objects created with `new`, and `this.` calls.
- All of these are ignored silently.

### T5. Self-invocation fixes, ranked — `MK`
1. Move the method to a separate bean.
2. Self-inject (`@Lazy` or `ObjectProvider`).
3. `AopContext.currentProxy()` with `exposeProxy`.
4. Programmatic APIs instead of the annotation.
5. AspectJ weaving.

### T6. Inspecting proxies — `MK`
- `AopUtils`, `AopProxyUtils.ultimateTargetClass`, `Advised.getAdvisors()`, `AopTestUtils`.
- `[3.x]` The proxy class-name marker changes from `$$EnhancerBySpringCGLIB$$` to `$$SpringCGLIB$$`.

### T7. Proxy-based annotations — `MK`
- `@Transactional`, `@Cacheable`, `@Async`, `@Retryable`, `@PreAuthorize`, `@Validated`, `@Timed` / `@Observed`.
- Several annotations on one bean produce one proxy with many advisors, not nested proxies.

### T8. Proxy cost — `GTH`
- Measure with JMH before blaming proxies.

### T9. Build it: mini proxy framework — `MK`
- An interceptor chain, plus a reproduction of the self-invocation bug.

### Traps
- Reading state off a CGLIB proxy's fields returns defaults.
- Injecting a JDK proxy by its concrete class → `BeanNotOfRequiredTypeException`.

---

## M11. Spring AOP

### T1. Vocabulary — `MK`
- Aspect, join point, advice, pointcut, introduction, target, proxy, weaving.
- Spring AOP supports method-execution join points only.

### T2. @AspectJ annotation style — `MK`
- `spring-boot-starter-aop` (`spring.aop.auto`) or `@EnableAspectJAutoProxy`.
- Aspects are beans.

### T3. Advice types — `MK`
- `@Before`, `@AfterReturning`, `@AfterThrowing`, `@After`, `@Around`.
- The `JoinPoint` and `ProceedingJoinPoint` APIs, including `proceed(args)` to rewrite arguments.

### T4. Pointcut language — `MK`
- Supported designators: `execution`, `within`, `this`, `target`, `args`, `@annotation`, `@within`, `@target`, `@args`, `bean()`.
- The `execution(...)` grammar and its wildcards.
- `this` vs `target`.
- Combining pointcuts, named `@Pointcut` methods, parameter binding and `argNames`.

### T5. Ordering — `MK`
- Precedence within one aspect; across aspects via `@Order`.
- Ordering against the built-in advisors: cache outside the transaction, retry outside the transaction.

### T6. Domain aspects — `MK`
- Audit trail for account status changes, method timing, exception translation, an idempotency guard.
- `@Around` pitfalls: swallowing exceptions, or forgetting to return the result.

### T7. Introductions — `GTH`
- `@DeclareParents` and `DelegatingIntroductionInterceptor`.

### T8. Low-level AOP API — `MK`
- `Pointcut`, `ClassFilter`, `MethodMatcher` (static vs runtime matching).
- The AOP Alliance `MethodInterceptor`, and the before / after-returning / throws advice interfaces.
- `Advisor` and `DefaultPointcutAdvisor`; `AnnotationMatchingPointcut`, `AspectJExpressionPointcut`, `ComposablePointcut`.
- Programmatic `ProxyFactory`, and inspecting or mutating a live proxy through `Advised`.

### T9. TargetSource — `GTH`
- Singleton, hot-swappable, pooling, prototype and thread-local target sources.
- `SimpleBeanTargetSource`: the one behind scoped proxies.

### T10. Auto-proxy creators — `MK`
- `AbstractAutoProxyCreator.wrapIfNecessary`.
- The implementations:
  - `DefaultAdvisorAutoProxyCreator`, `BeanNameAutoProxyCreator`.
  - `InfrastructureAdvisorAutoProxyCreator` (used for transactions).
  - `AnnotationAwareAspectJAutoProxyCreator` (used for `@Aspect`).
- `GTH` `ProxyFactoryBean`.

### T11. Internals — `GTH`
- `ReflectiveMethodInvocation.proceed()` and `ExposeInvocationInterceptor`.
- The per-method advisor-chain cache and advice adapters.
- The JDK vs CGLIB invocation paths.

### T12. AspectJ weaving — `GTH`
- Compile-time weaving (ajc / aspectj-maven-plugin) and load-time weaving (`@EnableLoadTimeWeaving`, the `-javaagent` flag).
- `spring-aspects` (`@Configurable`, `AnnotationTransactionAspect`) and `AdviceMode.ASPECTJ`.
- Comparison table: Spring AOP vs AspectJ.

### Traps
- An unsupported pointcut designator fails at startup.
- A broad `execution(* *(..))` pointcut proxies infrastructure beans too.
- An `@Around` advice without `proceed()` silently skips the method.
