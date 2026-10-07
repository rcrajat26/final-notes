# 06 — Spring Boot Core

## M20. What Boot Adds

### T1. The same service three ways — `MK` `[P→S→B]`
- `client-service` in plain Java with embedded Tomcat started by hand.
- In plain Spring: `DispatcherServlet` registration and manual config.
- In Boot.

### T2. @SpringBootApplication decomposed — `MK`
- It is `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan` (with `TypeExcludeFilter`).
- Its attributes.

### T3. Starters and dependency management — `MK`
- A starter is a POM that aggregates dependencies; the auto-config lives elsewhere.
- The `spring-boot-dependencies` BOM; using the parent vs importing the BOM.
- Overriding a managed version via a property.

### T4. SpringApplication — `MK`
- The `run()` sequence and `WebApplicationType` detection.
- Customization: the builder, listeners, `spring.main.*` properties.

### T5. run() internals — `MK`
- `prepareEnvironment` (where `EnvironmentPostProcessor`s run) → create context → `prepareContext` → refresh → runners.
- Failure handling and `FailureAnalyzer`s.

### T6. Embedded servers — `MK`
- Tomcat by default; swapping to Jetty or Undertow; `ServletWebServerFactory`.
- The server is created during refresh but only starts accepting traffic at the end.

### T7. Runners — `MK`
- `ApplicationRunner` vs `CommandLineRunner`: ordering, and failures stop the app.

### T8. DevTools — `GTH`

### T9. Version baselines — `MK`
- Comparison table: 2.7 vs 3.x.
- `[4.x]` Modular starters.

### Traps
- Pinning library versions yourself instead of using the BOM.
- Adding a starter and expecting behaviour without the matching properties.

---

## M21. Auto-configuration

### T1. Configuration three ways — `MK` `[P→S→B]`
- An `ObjectMapper` / `DataSource` configured by hand → in a `@Configuration` class → auto-configured and backing off when you define your own.

### T2. The mechanism — `MK`
- `AutoConfigurationImportSelector` is a `DeferredImportSelector`, so it runs after your configuration.
- Candidates are listed in `AutoConfiguration.imports` (2.7+).
- `spring.factories` was deprecated for this in 2.7; `[3.x]` removed in 3.0.
- `@AutoConfiguration` (2.7+).

### T3. Conditions — `MK`
- The key conditions; `@ConditionalOnProperty`'s `matchIfMissing`.
- `@ConditionalOnBean` is only reliable inside auto-configuration classes.
- The return-type trap: `@ConditionalOnMissingBean` matches on the `@Bean` method's declared return type.
- `[3.x]` New conditions: `@ConditionalOnThreading`, `@ConditionalOnBooleanProperty`; stricter `.enabled` parsing in 3.5.

### T4. Ordering — `GTH`
- `@AutoConfigureBefore` / `After` / `Order`; the sorter; metadata pre-filtering.

### T5. Diagnosis — `MK`
- The `--debug` conditions report and `/actuator/conditions`: reading positive and negative matches.
- Excluding an auto-configuration.

### T6. Customizing vs replacing — `MK`
- `*Customizer` beans (Jackson, web server, `RestTemplate`) as the intended extension point.
- Defining your own bean so the auto-config backs off.

### T7. Writing a custom starter — `MK`
- Naming; the autoconfigure + starter split; the imports file.
- Properties with generated metadata; conditions.
- Example: a client/account audit starter.

### T8. Testing auto-configuration — `MK`
- `ApplicationContextRunner` and `FilteredClassLoader`.

### T9. Build it — `MK`
- A mini auto-configuration mechanism with back-off.

### T10. `[4.x]` Modularized autoconfigure jars

### Traps
- `@EnableWebMvc` switches off Boot's MVC auto-config.
- `@ConditionalOnBean` used in user configuration is order-dependent.

---

## M22. Externalized Configuration

### T1. Precedence — `MK`
- The full 15-source list.
- Trap: a stale environment variable overrides the yml you just edited.

### T2. Config data files — `MK`
- `application.properties` / `.yml`; default locations; profile-specific files; `.properties` beats `.yaml` in the same location.
- `spring.config.name`, `location`, `additional-location`.
- The Boot 2.4 config-data model.

### T3. spring.config.import and multi-document files — `MK`
- `optional:` imports; `GTH` `configtree:`.
- Multi-document files with `spring.config.activate.on-profile`, which replaced the old `spring.profiles` key.

### T4. @ConfigurationProperties — `MK`
- Binding modes:
  - JavaBean (setter) binding.
  - Constructor binding: `@ConstructorBinding` goes on the type in 2.x, on the constructor in `[3.x]`.
  - Records.
- Nested types, collections and maps (a list is replaced wholesale, not merged).
- `@DefaultValue`; enabling via `@EnableConfigurationProperties` or `@ConfigurationPropertiesScan`.

### T5. Relaxed binding — `MK`
- The equivalent forms (kebab, camel, underscore, env var) and the env-var mapping rules.
- `@Value` doesn't get relaxed binding.

### T6. Validation and metadata — `MK`
- `@Validated` properties classes that fail fast at startup.
- The configuration processor for IDE metadata.

### T7. Conversion in binding — `MK`
- `Duration`, `DataSize`, `@DurationUnit`; custom converters.

### T8. Secrets and environments — `MK`
- Profiles for behaviour, environment for values.
- Secret managers; `GTH` AWS Parameter Store / Secrets Manager via Spring Cloud AWS.

### T9. Debugging — `MK`
- `/actuator/env` (winning and shadowed values) and `/actuator/configprops` (what actually bound).

### T10. Binder API — `GTH`
- Programmatic `Binder` usage, and a mini relaxed binder as a build-it.

### T11. Runtime refresh — `GTH`
- `@RefreshScope`; see M31.

### Traps
- A missing value silently filled by a `@Value` default.
- A list in a profile file replaces the base list instead of merging.

---

## M23. Logging

### T1. Logging three ways — `MK` `[P→S→B]`
- `System.out` / JUL → SLF4J with Logback → Boot's logging system.

### T2. Facades and bridges — `MK`
- SLF4J; Spring's own `spring-jcl` bridge; avoiding duplicate bindings.

### T3. Boot logging configuration — `MK`
- `logging.level`, `logging.group`, `logging.file.*`, `logging.pattern.*`; rolling-file properties.
- `logback-spring.xml` with `<springProfile>` and `<springProperty>`.

### T4. Switching to Log4j2 — `GTH`

### T5. MDC — `MK`
- A filter that puts `clientId` / `accountId` into the MDC.
- Propagating MDC to `@Async` threads with a `TaskDecorator`.

### T6. Structured JSON logging — `MK`
- In 2.7: `logstash-logback-encoder`.
- `[3.x]` Built-in structured logging (3.4+, ECS / Logstash / GELF formats).

### T7. Runtime log levels — `MK`
- Changing levels live via `/actuator/loggers`.

### T8. Correlation with traces — see M41

### Traps
- Logging PII (email, phone) on a regulated platform.
- Building log messages by string concatenation instead of placeholders.
- `logback.xml` vs `logback-spring.xml`.
