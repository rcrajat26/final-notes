# Annotations
## What an annotation actually is
- An annotation is a form of metadata that can be added to Java code (classes, methods, variables, parameters, or even another annotation, etc.) to provide additional information to the compiler or runtime.
- It carries no direct executable behavior of its own. It's purely a structured, machine-readable tag.
- Annotations do not directly affect program semantics but can influence how the code is processed by tools, frameworks, or the Java compiler itself.
- An annotation with nothing reading it does nothing at all — it's inert metadata sitting in the bytecode (or sometimes not even there — see retention below).
- Syntactically, an annotation is itself a special kind of interface:
```java
public @interface Deprecated { 

}
```

- `@interface` (not `interface`) signals to the compiler "this defines an annotation type." 
- Annotation types implicitly extend `java.lang.annotation.Annotation` and cannot themselves extend anything else or be extended.

### Built-in Annotations
| Annotation             | Target                  | Purpose                                                                                                                                                                                              |
|------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `@Override`            | method                  | Compile-time-only assertion that this method actually overrides a superclass/interface method — misspelling a method name with `@Override` present turns a silent bug into a compile error           |
| `@Deprecated`          | class/method/field/etc. | Marks an element as discouraged for future use; `@Deprecated(since="...", forRemoval=true)` (Java 9+) adds structured version metadata                                                               |
| `@SuppressWarnings`    | class/method/field      | Tells the compiler to suppress specific warning categories (`"unchecked"`, `"deprecation"`, etc.) for the annotated scope — common when deliberately using a generic-unsafe cast you know is correct |
| `@FunctionalInterface` | interface               | Compile-time assertion that an interface has exactly one abstract method                                                                                                                             |
| `@SafeVarargs`         | method/constructor      | Suppresses heap-pollution warnings for varargs methods using generics, asserting the method body doesn't do anything unsafe with the varargs array                                                   |
### Writing a custom annotation
```java
public @interface AuditLog {
    String value() default "INFO";
    boolean sensitive() default false;
}
```
- Members are declared method-style (no body, no parameters) — these are the annotation's elements.
`value()` is special: if it's the only element being set, you can omit the name: `@AuditLog("WARN")` instead of `@AuditLog(value = "WARN")`.
- default makes an element optional; without it, every use of the annotation must supply that element explicitly.
- Allowed element types are restricted: primitives, String, Class, enums, annotations, and arrays of any of those — not arbitrary objects. 
- This restriction exists because annotation values must be resolvable as compile-time constants, storable directly in the class file.

Ex:
```java
public class AccountService {
    @AuditLog(value = "WARN", sensitive = true)
    public void suspendAccount(Account account) { ... }
}
```
> Usage of this is demonstrated in the miscellaneous section 

## Meta-annotations — annotating the annotation
These four control how a custom annotation behaves, and are themselves just annotations applied to @interface declarations:

### @Target
`@Target` — restricts which kinds of program elements the annotation can legally be placed on, checked by the compiler.

```java
@Target({ElementType.METHOD, ElementType.FIELD})
public @interface AuditLog { ... }
```

- `ElementType` values: `TYPE` (class/interface/enum), `FIELD`, `METHOD`, `PARAMETER`, `CONSTRUCTOR`, `LOCAL_VARIABLE`, `ANNOTATION_TYPE`, `PACKAGE`, `TYPE_PARAMETER` (Java 8+, generic type parameters), `TYPE_USE` (Java 8+, any type usage, including inside generics/casts — what enables tools like nullability-checking frameworks to annotate List<@NonNull String>). 
Omitting @Target entirely means the annotation is legal almost everywhere.

### @Retention
`@Retention` — controls whether the annotation is present in the compiled class file and/or available at runtime via reflection.
```java
@Retention(RetentionPolicy.RUNTIME)
public @interface AuditLog { ... }
```

**Retention Policy**:

| RetentionPolicy | Survives to                                              | Who can see it                                                                                               |
|-----------------|----------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| `SOURCE`        | Discarded by the compiler                                | Only annotation processors running at compile time (e.g. `@Override`, Lombok-style generators)               |
| `CLASS`         | Written into the `.class` file                           | Bytecode tools, but not visible via normal runtime reflection                                                |
| `RUNTIME`       | Written into the `.class` file **AND** loaded by the JVM | Visible via reflection at runtime (`getAnnotation(...)`) — what frameworks like Spring/JPA/Jackson depend on |

- `RetentionPolicy` values: 
  - `SOURCE` (discarded by the compiler, never in the class file), 
  - `CLASS` (in the class file but not loaded at runtime), 
  - `RUNTIME` (in the class file and available at runtime via reflection). 
- Default is CLASS if you don't specify @Retention. 
- If you want to read the annotation at runtime (e.g., for a framework that inspects methods and applies behavior based on annotations), you must use RUNTIME.

### @Repeatable 
@Repeatable (Java 8+) — allows the same annotation type to be applied more than once to the same element, by designating a containing annotation that implicitly wraps multiple instances:
```java
@Repeatable(Schedules.class)
public @interface Schedule { String day(); }

@Schedule(day = "MON")
@Schedule(day = "WED")
public void runJob() { ... }
```

Without @Repeatable, applying the same annotation twice to one element is a compile error.

---

# Miscellaneous
## How annotations are actually read
Annotations only do something when a processor picks them up. There are three common ways this happens:
- At runtime, via reflection (needs @Retention(RetentionPolicy.RUNTIME) on the annotation)
- At runtime, via a framework like Spring AOP, which intercepts the method call
- At compile time, via an annotation processor

### Reflection — the `RUNTIME`-retention use case
```java
Method m = AccountService.class.getMethod("suspendAccount", Account.class);
AuditLog annotation = m.getAnnotation(AuditLog.class);
if (annotation != null) {
    System.out.println(annotation.value()); // "WARN"
}
```

- `getAnnotation(Class)` returns `null` (not an exception) if the annotation isn't present at all, or if it's present but not `RUNTIME`-retained — from the caller's perspective, those two cases are indistinguishable without checking source.

### Annotation processors — the `SOURCE`-retention use case
- Tools like Lombok, Dagger, and MapStruct hook into the compiler via the `javax.annotation.processing` API, reading `SOURCE`-retained annotations during compilation and generating new source files or bytecode.
- As a side effect — the annotation itself never needs to survive to runtime because its entire job is done at compile time.

## How it works in real projects
- In practice you rarely write the reflection yourself. A framework like Spring does it for you with an aspect:
```java
@Aspect
@Component
public class AuditAspect {

    @Around("@annotation(auditLog)")
    public Object audit(ProceedingJoinPoint pjp, AuditLog auditLog) throws Throwable {
        String level = auditLog.value();
        boolean sensitive = auditLog.sensitive();

        log(level, "Calling " + pjp.getSignature()
            + (sensitive ? " [args hidden]" : " with " + Arrays.toString(pjp.getArgs())));

        return pjp.proceed();   // run the real method
    }
}
```

## Spring Boot annotation example @Cacheable
- A good pick is **`@Cacheable`**. It's simple to explain, and you can *see* its effect: the method body is skipped on repeat calls. 
- It works the same way as `@Transactional`, `@Async`, and `@Retryable`, so understanding it unlocks all of them.

### The example
```java
@SpringBootApplication
@EnableCaching                      // switch the feature on
public class App { ... }
```

```java
@Service
public class ProductService {

    @Cacheable("products")          // cache name
    public Product getProduct(Long id) {
        System.out.println("Hitting the database for " + id);
        return repository.findById(id).orElseThrow();
    }
}
```

Calling it:

```java
productService.getProduct(5L);  // prints "Hitting the database for 5"
productService.getProduct(5L);  // prints nothing, result comes from cache
productService.getProduct(7L);  // prints again (different argument = different key)
```

Nothing in `getProduct` mentions caching, yet the second call never executes the method body. That behavior comes from the annotation.

## How it gets picked up
 **who reads the annotation?**
**1. Startup: Spring scans your beans.** `@EnableCaching` registers a few infrastructure components, including a `BeanPostProcessor`. As Spring creates each bean (like `ProductService`), this post-processor inspects its methods and asks, "does any method carry `@Cacheable`, `@CacheEvict`, etc.?" It does this using reflection, which is why the annotation has runtime retention.

**2. Spring wraps the bean in a proxy.** If it finds a match, Spring doesn't put your plain `ProductService` object into the application context. It puts a **proxy**, a generated subclass (via CGLIB) that looks like `ProductService` but intercepts calls.

**3. Call time: the proxy intercepts.** Anyone who injects `ProductService` actually receives the proxy:

```
Caller → [Proxy] → (cache interceptor) → real ProductService.getProduct()
```

The interceptor does roughly this:

```java
// pseudo-code of what Spring does for you
Object key = buildKey(method, args);          // e.g. 5L
Object cached = cache("products").get(key);
if (cached != null) return cached;            // skip the real method

Object result = realMethod.invoke(args);      // run your code
cache("products").put(key, result);           // store it
return result;
```

So the annotation's values (`"products"` here) are read by the interceptor and used to decide which cache to look in.

## What the annotation attributes control

```java
@Cacheable(
    value = "products",                 // which cache
    key = "#id",                        // SpEL expression for the cache key
    condition = "#id > 0",              // only cache if this is true
    unless = "#result == null"          // don't cache if this is true
)
```

These are the equivalent of `value` and `sensitive` in your `@AuditLog`: just data that the interceptor reads and acts on.

## Gotchas (all caused by the proxy design)
- **Must be a Spring-managed bean.** `new ProductService().getProduct(5L)` bypasses the proxy, so no caching.
- **Self-invocation doesn't work.** If another method *inside* `ProductService` calls `this.getProduct(5L)`, it goes straight to the real object, not through the proxy, so the cache is skipped.
- **Needs to be public and non-final** (for the common proxy setups), since the proxy has to override the method.
- **Default cache is in-memory.** Spring Boot falls back to a simple `ConcurrentHashMap`. For real use you'd plug in Redis, Caffeine, etc., and the annotation code stays unchanged.

## Connecting it back to your `@AuditLog`

|                    | `@AuditLog` (your example) | `@Cacheable`                          |
|--------------------|----------------------------|---------------------------------------|
| Annotation         | Just metadata              | Just metadata                         |
| Who reads it       | An aspect you write        | `CacheInterceptor`, written by Spring |
| How it's triggered | Proxy intercepts the call  | Proxy intercepts the call             |
| What it does       | Logs                       | Caches                                |

`@Cacheable` is essentially a prebuilt version of the `AuditAspect` I showed earlier. Spring ships the aspect, and you just supply the annotation.


