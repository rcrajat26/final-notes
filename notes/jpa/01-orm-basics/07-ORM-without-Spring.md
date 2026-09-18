Pom.xml: 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.quizstakes</groupId>
    <artifactId>quizstakes-plain-hibernate</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.hibernate.orm</groupId>
            <artifactId>hibernate-core</artifactId>
            <version>6.6.2.Final</version>
        </dependency>
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
            <version>5.1.0</version>
        </dependency>
        <dependency>
            <groupId>org.hibernate.orm</groupId>
            <artifactId>hibernate-hikaricp</artifactId>
            <version>6.6.2.Final</version>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <version>2.2.224</version>
        </dependency>
    </dependencies>
</project>
```

hibernate.properties: 
```properties
hibernate.connection.provider_class=org.hibernate.hikaricp.internal.HikariCPConnectionProvider

hibernate.hikari.jdbcUrl=jdbc:h2:mem:quizstakes;DB_CLOSE_DELAY=-1
hibernate.hikari.username=sa
hibernate.hikari.password=
hibernate.hikari.maximumPoolSize=10

hibernate.hbm2ddl.auto=update
hibernate.show_sql=true
```

Client entity:
```java
@Entity
@Getter @Setter @NoArgsConstructor
@Table(name = "client")
public class Client {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    @OneToMany(mappedBy = "client", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Account> accounts = new ArrayList<>();

    public void addAccount(Account account) {
        accounts.add(account);
        account.setClient(this);
    }
}
```

Account entity:
```java
@Entity
@Getter @Setter @NoArgsConstructor
@Table(name = "account")
public class Account {

    public enum AccountType { IGCFD, IGSTK }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)
    @Column(name = "account_type", nullable = false)
    private AccountType accountType;

    @Column(nullable = false)
    private BigDecimal balance;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "client_id", nullable = false)
    private Client client;
}
```

Main class:
```java
public class Main {

    public static void main(String[] args) {

        // --- steps 1 + 2: registry reads hibernate.properties, pool fills here ---
        StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
                .configure() // loads hibernate.properties from classpath
                .build();

        SessionFactory sessionFactory;
        try {
            sessionFactory = new MetadataSources(registry)
                    .addAnnotatedClass(Client.class)
                    .addAnnotatedClass(Account.class)
                    .buildMetadata()
                    .buildSessionFactory();
        } catch (Exception e) {
            StandardServiceRegistryBuilder.destroy(registry);
            throw e;
        }

        // --- step 3: borrow — Session wraps one physical connection ---
        try (Session session = sessionFactory.openSession()) {

            // --- step 4: mark the atomic block ---
            Transaction tx = session.beginTransaction();

            try {
                // --- step 5: statements, buffered in the persistence context ---
                Client client = new Client("Rajat", "rajat@example.com");
                client.addAccount(new Account(Account.AccountType.IGCFD, BigDecimal.ZERO));
                client.addAccount(new Account(Account.AccountType.IGSTK, BigDecimal.ZERO));

                session.persist(client); // transient -> managed; nothing sent to DB yet

                // --- step 6: commit — flush + physical COMMIT + locks released ---
                tx.commit();

                System.out.println("Saved client id=" + client.getId()
                        + " with " + client.getAccounts().size() + " accounts");

            } catch (RuntimeException e) {
                tx.rollback();
                throw e;
            }

        } // --- step 7: Session.close() returns the connection to the Hikari pool ---
        finally {
            sessionFactory.close();       // shuts down the SessionFactory
            StandardServiceRegistryBuilder.destroy(registry); // and the Hikari pool with it
        }
    }
}
```

Note: 
- Everything Spring quietly did for you in approaches 1 and 2 is spelled out here: 
- the beginTransaction()/commit()/rollback() triplet is @Transactional; 
- the try/finally around the Session is what the AOP proxy guarantees automatically; 
- and sessionFactory.close() / StandardServiceRegistryBuilder.destroy(registry) is the explicit shutdown that an application context would otherwise trigger for you on exit.


---
### Miscellaneous.
(Pasted as is, needs refinement.)
1. What **Spring (the core framework)** contributes — separate from Boot's auto-config and separate from what JPA/Hibernate itself does:

2. **Dependency injection / IoC container** — beans (`DataSource`, `EntityManagerFactory`, services, repositories) are constructed and wired together by the container, not by your code calling `new` everywhere.

3. **`@Transactional` + `PlatformTransactionManager` abstraction** — declarative transaction demarcation. You write one annotation; Spring's AOP proxy handles begin/commit/rollback around the method.

4. **Thread-bound resource management** — `TransactionSynchronizationManager` binds one `EntityManager`/`Connection` per thread per transaction, so every repository call inside a `@Transactional` method transparently shares the same one instead of you passing it around manually.

5. **Guaranteed cleanup via AOP proxy** — the proxy wraps your method the way a `try/finally` would, so commit/rollback and connection release happen even if you never write that boilerplate yourself. (This is also *why* self-invocation breaks it — no proxy in the call path.)

6. **Exception translation** — low-level JDBC `SQLException`s and JPA `PersistenceException`s get converted into Spring's unified `DataAccessException` hierarchy, so calling code doesn't need to know which provider threw what.

7. **`JpaRepository` / Spring Data abstraction** (when you add Spring Data JPA) — generates the DAO implementation at startup from an interface; you get `save()`, `findById()`, derived query methods, etc., without writing implementation code.

8. **Lifecycle management** — the `ApplicationContext` owns bean creation and destruction order, so pool startup/shutdown, `EntityManagerFactory` startup/shutdown, and thread-pool/resource cleanup all happen in a controlled sequence tied to app startup/shutdown, instead of you managing it by hand (as in the no-Spring example).

9. **Configuration abstraction** — one consistent way (`@Configuration`/`@Bean`, or Boot's `application.yml`) to wire alternative implementations (swap Hikari for another pool, Hibernate for EclipseLink) without touching business logic.

Everything else — entity mapping, dirty checking, flush timing, SQL generation, locking — is Hibernate/JPA, not Spring. Spring's job is purely the *wiring, lifecycle, and transaction orchestration* around it.