# 01 — Foundations

## M01. History of Java Backend Development
Goal: understand why Spring exists, and what each later layer abstracts away.

### T1. Before Java on the server — `MK`
- CGI: one process per request, Perl/C scripts, and why it didn't scale.
- Java's move from applets to the server.

### T2. Servlets and JSP — `MK`
- The Servlet API (1997):
  - The container and its thread-per-request model.
  - The servlet lifecycle: `init` / `service` / `destroy`.
- JSP; Model 1 vs Model 2 (MVC); Struts 1 as the first popular MVC framework.
- What a servlet developer hand-wrote: routing, parameter parsing, binding, object wiring, transactions.

### T3. J2EE and EJB 2.x — `MK`
- J2EE 1.2 (1999):
  - Application servers: WebLogic, WebSphere, JBoss.
  - Platform services: JNDI, JTA, JMS.
- The EJB 2.x model:
  - Home, remote and local interfaces.
  - Session, entity and message-driven beans.
  - Deployment descriptors.
- Container-managed transactions (CMT), and the "rollback only on runtime exceptions" convention Spring later inherited.
- Pain points:
  - Heavyweight and untestable outside the container.
  - JNDI lookup boilerplate.
  - Slow deploy cycles.
  - Poor entity-bean performance.

### T4. The POJO movement — `MK`
- The "J2EE without EJB" idea, and lightweight containers (PicoContainer, Avalon, HiveMind).
- Fowler's 2004 article that named "Dependency Injection".

### T5. Birth of Spring — `MK`
- Rod Johnson's *Expert One-on-One J2EE Design and Development* (2002) and its framework code.
- Juergen Hoeller, open-sourcing, and Spring 1.0 (2004).
- The core thesis:
  - Plain Java objects (POJOs), wired externally.
  - Declarative services via proxies (transactions without EJB).
  - Portable abstractions such as `DataAccessException`.

### T6. Spring Framework evolution — `MK`
- Version timeline:
  - 2.0: XML namespaces and AspectJ integration.
  - 2.5: annotation-driven configuration.
  - 3.0: Java config, SpEL, REST support.
  - 4.0: Java 8 and `@Conditional`.
  - 5.0: reactive WebFlux.
  - 6.0: Jakarta, AOT, Java 17 baseline.
  - `[4.x]` 7.0: null-safety, API versioning, built-in resilience.
- Spring's influence on Java EE 5/6: EJB 3, JPA (via Hibernate), CDI.
- `GTH` Stewardship: SpringSource → VMware → Pivotal → VMware → Broadcom.

### T7. Spring Boot — `MK`
- The 2012 request for container-less Spring web apps, leading to Boot 1.0 (2014).
- What Boot brought: opinionated defaults, starters, auto-configuration, embedded servers, production features.
- The context it arose in: microservices, the 12-factor app, Dropwizard.
- Boot milestones:
  - 2.0 (2018): reactive stack, Micrometer.
  - 2.7 (2022): the last 2.x line.
  - 3.0 (2022): Jakarta, native images.
  - `[4.x]` 4.0 (2025).

### T8. Java EE → Jakarta EE — `MK`
- Transfer from Oracle to Eclipse (2017), the trademark issue, and the `javax` → `jakarta` rename (Jakarta EE 9).
- `[3.x]` Why most of the 2.7 → 3.x migration is this rename.

### T9. Today's landscape — `GTH`
- Quarkus, Micronaut and Helidon: build-time DI versus Spring's runtime container.
- Why Spring is still dominant.

### T10. Map of what Spring abstracts — `MK`
- One table: each hand-written plain-Java concern → its Spring abstraction → its Boot automation.
- Concerns covered: wiring, configuration, transactions, web, server, operations.

---

## M02. IoC and Dependency Injection

### T1. The problem in plain Java — `MK` `[P→S→B]`
- `ClientService` constructing `AccountService` and repositories with `new`: coupling, testability and lifetime problems.
- A manual composition root in `main()`; the factory and service-locator patterns.

### T2. IoC and DI defined — `MK`
- What is actually inverted: construction and lifetime of the object graph.
- DI vs service locator vs dependency lookup (`getBean`).

### T3. The container — `MK`
- `BeanFactory` vs `ApplicationContext`: the feature table (post-processor auto-registration, events, `MessageSource`, resources, `Environment`).
- Context implementations:
  - `AnnotationConfigApplicationContext`, `GenericApplicationContext`.
  - The web contexts (servlet and reactive).
  - `GTH` XML contexts, for reading legacy code.
- Bootstrapping a context by hand, and closing it with try-with-resources.
- Module map: core, beans, context, aop, tx, web, webmvc, webflux, test.

### T4. Injection styles — `MK`
- Constructor injection:
  - Why it's the default: `final` fields, no half-built objects, container-free tests, visible pressure to keep classes small.
  - With a single constructor, no `@Autowired` is needed (since 4.3).
- Setter injection, for optional or reconfigurable dependencies.
- Field injection: done via reflection, can't be `final`, hides dependency cycles.
- Method injection on arbitrary methods.

### T5. Wiring compared — `MK` `[P→S→B]`
- Client and account services wired by hand, then with a Spring context, then with Boot.

### T6. Build it: mini container — `MK`
- Registry, singleton cache, constructor resolution, cycle detection.

### Traps
- Calling `getBean()` in application code is a service-locator regression.
- Field injection masks circular dependencies.
- `@Autowired` on a `static` field stays `null`, silently.

---

## M03. Bean Definitions and Configuration Sources

### T1. The BeanDefinition model — `MK`
- A bean is a definition plus the instances created from it.
- Key properties: class, scope, lazy, primary, dependsOn, init/destroy methods, factory method.
- Naming:
  - `@Component("x")`, or the `@Bean` method name.
  - The default decapitalisation rule, including the two-leading-capitals exception (`URLHandler` stays `URLHandler`).
  - Aliases.
- Overriding:
  - `spring.main.allow-bean-definition-overriding` is `false` since Boot 2.1.
  - The resulting `BeanDefinitionOverrideException`.
- `GTH` Bean roles (`ROLE_APPLICATION` / `SUPPORT` / `INFRASTRUCTURE`) and merged bean definitions.

### T2. Configuration sources — `MK`
- XML (`GTH`, for literacy).
- Component scanning.
- `@Configuration` / `@Bean`.
- Programmatic `registerBean(Class, Supplier)`.
- `[4.x]` The new `BeanRegistrar` API.

### T3. Component scanning — `MK`
- `@ComponentScan` attributes; filter types; `useDefaultFilters=false`.
- `@SpringBootApplication` scans its own package and below: the main-class placement rule.

### T4. Stereotypes — `MK`
- `@Component` and `@Service` have no behaviour of their own.
- `@Repository` adds exception translation into the `DataAccessException` hierarchy.
- `@Controller`, `@RestController` and `@Configuration` each add real behaviour.

### T5. Meta-annotations — `MK`
- Composed annotations, `@AliasFor`, `MergedAnnotations`.
- Annotations must have RUNTIME retention for Spring to see them.
- Exercise: write a composed `@BusinessService`.

### T6. The @Import family — `MK`
- What `@Import` can import:
  - Plain components and configuration classes.
  - `ImportSelector`.
  - `DeferredImportSelector`: runs after user configuration, which is how auto-configuration backs off.
  - `ImportBeanDefinitionRegistrar`: how Feign clients and Spring Data repositories get registered.
- The `@Enable*` convention; `AdviceMode.PROXY` vs `ASPECTJ`.
- `GTH` `@ImportResource`.

### T7. FactoryBean — `GTH`
- `getObject` / `getObjectType` / `isSingleton`.
- The `&` prefix retrieves the factory itself rather than its product.
- `FactoryBean` vs a `@Bean` method.

### Traps
- Main class in a leaf package → components not found.
- Switching `@Repository` to `@Component` silently loses exception translation.
- Two same-named classes in different packages → bean name collision.

---

## M04. Dependency Resolution and Injection Points

### T1. The resolution algorithm — `MK`
- Order: by type → qualifiers → `@Primary` → `@Priority` → parameter-name match.
- Reading the three failure types: `NoSuchBeanDefinitionException`, `NoUniqueBeanDefinitionException`, `UnsatisfiedDependencyException`.
- The name-match fallback depends on compiling with `-parameters`.

### T2. Disambiguation — `MK`
- `@Qualifier`, custom qualifier annotations, `@Primary`.
- `[3.x]` `@Fallback` (Framework 6.2).
- `autowireCandidate=false`.

### T3. Collections and generics — `MK`
- Injecting `List`, `Set` or `Map<String, T>` as a strategy registry (e.g. one handler per account type, IGCFD / IGSTK).
- `@Order` sorts injected collections but does *not* control startup order.
- Generic type parameters act as implicit qualifiers (`ResolvableType`).

### T4. Optional and lazy injection — `MK`
- `required=false` (method vs field semantics), `Optional`, `@Nullable`.
- `ObjectProvider` (`getIfAvailable`, `getIfUnique`, `stream`, `orderedStream`); the older `ObjectFactory` and `Provider`.
- `@Lazy` on an injection point (injects a lazy proxy) vs `@Lazy` on a definition (defers creation).
- `@DependsOn`.
- `GTH` `@Lookup` method injection.

### T5. Standard annotations — `GTH`
- `@Inject`, `@Named`, `@Resource` (matches by name first).
- `[3.x]` These move from `javax.inject` / `javax.annotation` to `jakarta.*`.

### T6. Built-in injectables and self-injection — `MK`
- Injectable without any bean definition: `ApplicationContext`, `Environment`, `BeanFactory`, `ApplicationEventPublisher`.
- A bean injecting itself: treated as the lowest-priority fallback candidate.

### T7. Circular dependencies — `MK`
- Constructor cycles always fail; field and setter cycles can resolve.
- Disabled by default since Boot 2.6 (`spring.main.allow-circular-references`).
- Fixes, ranked: extract a third bean, use an event, `ObjectProvider`, `@Lazy`.
- The mechanism behind this is covered in M09.

### Traps
- Ambiguity breaks after compiling without `-parameters`.
- `@Priority` on a `@Bean` method is ignored.
- `allow-circular-references=true` is not a fix.
