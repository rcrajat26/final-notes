# History of Java Backend Development
## Before Java on the server
- The first dynamic web pages were served through CGI. 
- For every request, the web server started a new operating-system process, usually a Perl or C script, which produced HTML and exited. 
- Process startup cost made this slow, and it fell apart under real traffic. 
- Java arrived in the mid-90s mostly as a browser technology (applets), but it soon moved to the server, where its threading model and portability were a better fit.

## Servlets and JSP
- The Servlet API (1997) fixed the CGI cost. A servlet container (Tomcat is the famous one) stays running and keeps your servlet objects alive. 
- Each request is handled by a thread from a pool, so you get thread-per-request instead of process-per-request. 
- The lifecycle is simple: the container calls init once, service for every request, and destroy at shutdown.
- Servlets were low-level, though. You wrote HTML inside Java code, which led to JSP and then to the realization that mixing both was a mess. 
- The community moved from "Model 1" (JSP does everything) to "Model 2" (a servlet controller picks the logic, and a JSP only renders), which is the MVC pattern. 
- Struts 1 was the first popular framework built around it.
- Even so, a servlet developer hand-wrote almost everything else: routing URLs to code, parsing parameters, converting strings to objects, wiring objects together, and managing transactions. 
- Almost every later piece of Spring exists to remove one of those chores.

## J2EE and EJB 2.x
- Enterprises needed more than a web tier. They needed distributed transactions, messaging, and security. 
- J2EE 1.2 (1999) answered this with a specification that application servers (WebLogic, WebSphere, JBoss) implemented, along with platform services: JNDI for lookups, JTA for transactions, and JMS for messaging.
- The centerpiece was the Enterprise JavaBean. EJB 2.x had session beans (business logic), entity beans (persistence), and message-driven beans. 
- Each bean needed home and remote or local interfaces plus XML deployment descriptors. 
- In exchange, you got container-managed transactions (CMT): you declared that a method was transactional and the container handled begin and commit. 
- One convention from that era is worth remembering because Spring inherited it: a transaction rolled back automatically on runtime exceptions but not on checked ones.
- The cost was heavy:
  - Beans could not run outside the container, so unit testing was nearly impossible.
  - Every collaborator needed JNDI lookup boilerplate.
  - Deploy cycles were slow.
  - Entity beans performed badly.

## The POJO movement
- The backlash was the idea of "J2EE without EJB": keep business code as plain Java objects (POJOs) and let a lightweight container supply the services. 
- Early containers like PicoContainer, Avalon, and HiveMind explored this. 
- In 2004 Martin Fowler wrote the article that gave the technique its name, Dependency Injection: instead of an object looking up or building its collaborators, the container hands them in.

## The birth of Spring
- Rod Johnson's 2002 book Expert One-on-One J2EE Design and Development argued that most EJB use was unnecessary, and shipped with framework code that became Spring's seed. 
- Juergen Hoeller joined, the code was open-sourced, and Spring 1.0 came out in 2004.
- Its thesis has three parts, and you will see them throughout the course:
  - POJOs wired externally. Your classes know nothing about Spring, and the container assembles them.
  - Declarative services through proxies. A transaction can wrap your method without EJB, because Spring puts a proxy in front of your object.
  - Portable abstractions. For example, DataAccessException gives you one exception hierarchy no matter which persistence technology sits underneath.

## How the Spring Framework evolved
- 2.0: XML namespaces (less verbose config) and AspectJ integration.
- 2.5: annotation-driven configuration (@Autowired, component scanning), moving config out of XML.
- 3.0: Java-based config, SpEL (the expression language), and REST support.
- 4.0: Java 8 support and @Conditional, which later powers Boot's auto-configuration.
- 5.0: the reactive stack, WebFlux.
- 6.0: the Jakarta namespace, AOT processing, and a Java 17 baseline.
- 7.0: null-safety, API versioning, and built-in resilience features.

## Spring Boot
- Even with annotations, a Spring app needed a lot of setup: choosing compatible library versions, configuring a data source, a view resolver, a transaction manager, and then packaging a WAR for an external server. 
- In 2012 a community member filed an issue asking for Spring web apps that could run without a container, and that led to Spring Boot 1.0 in 2014.
- Boot's contribution was convention over configuration:
  - Opinionated defaults for common setups.
  - Starters: one dependency pulls in a compatible set of libraries.
  - Auto-configuration: beans are created for you based on what is on the classpath.
  - Embedded servers: your app contains Tomcat, so java -jar is the deployment.
  - Production features: health checks, metrics, and externalized config.
- It arrived at the right time, alongside microservices, the 12-factor app methodology, and competitors like Dropwizard, which had proven that developers wanted this kind of packaging.
- Milestones to remember: 2.0 (2018) brought the reactive stack and Micrometer; 2.7 (2022) was the last 2.x line; 3.0 (late 2022) moved to Jakarta and added native-image support; 4.0 followed in 2025.

## Java EE becomes Jakarta EE
- Oracle handed Java EE to the Eclipse Foundation in 2017. Oracle kept the "Java" trademark, so the platform could not keep the javax.* package names for new development. 
- It was renamed Jakarta EE, and with Jakarta EE 9 the packages changed from javax.* to jakarta.*.
- This matters to you because Spring Framework 6 and Boot 3 adopted jakarta.*. 
- Most of the work of migrating from 2.7 to 3.x is this rename, plus the Java 17 requirement.

## Today's landscape (GTH)
- Quarkus, Micronaut, and Helidon are modern alternatives. Their main difference is that they do dependency injection at build time, while Spring wires at runtime through reflection and proxies. 
- That gives them faster startup and lower memory by default. Spring remains dominant because of its ecosystem, its huge pool of existing code and developers, and its answer to the startup gap, which is AOT and native images.

| Concern | Plain Java / servlet era | Spring abstraction | Boot automation |
|---|---|---|---|
| Wiring objects | `new`, factories, JNDI lookups | IoC container, DI | Component scanning from the main class, auto-configured beans |
| Configuration | Hand-parsed properties and XML | Environment, `@Configuration`, profiles | Externalised config with `application.yml`, env vars, sensible defaults |
| Transactions | Manual JDBC commit/rollback or EJB CMT | `@Transactional` via proxies, `PlatformTransactionManager` | Transaction manager auto-configured |
| Web layer | Hand-written servlets, parsing, binding | Spring MVC / WebFlux | `DispatcherServlet` and message converters pre-configured |
| Server | Install and configure Tomcat/app server, deploy a WAR | Servlet container integration | Embedded server in an executable jar |
| Operations | Custom health pages, ad-hoc logging | Cross-cutting support (events, caching) | Actuator, metrics, health probes, logging setup |




