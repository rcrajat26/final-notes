### How manual works without any tool
1. Open a database conversation — TCP connect, login, session setup. 5–50 ms. Far too expensive per query. 
2. So you pool. At startup, open ~10 connections and keep them alive. Nobody opens a connection after startup; they borrow one. 
3. A unit of work borrows one. It is now exclusively yours. Nine remain for everyone else. 
4. Mark the atomic block. autoCommit = false. Everything after this is provisional and invisible to others. 
5. Send statements. Each is a network round trip. The database takes and holds row locks on whatever you write. 
6. Commit or roll back. Commit makes it durable and visible, and releases the locks. 
7. Return the connection to the pool. Not close — return.

```md
Note: 
- "Marking the atomic block" means telling the database: everything I send from this point forward, until I explicitly say commit or rollback, is one indivisible unit — either all of it takes effect, or none of it does.
- By default, most DB drivers run in autoCommit = true — every single statement is its own tiny transaction, committed the instant it finishes.

```

```java
connection.setAutoCommit(false);   // <-- this is "marking" the start
// ... statements ...
connection.commit();               // or connection.rollback();
```

## How ORM does these steps:
### What happens at SpringBoot-application-startup?
- Run a Spring Boot app with `spring-boot-starter-data-jpa` dependency boots up
- Auto configuration sees Hikari on the classpath. Reads application.yml for data source properties constructs a hikari data source.
- Hikari eagerly establishes the minimum required connections using the JDBC driver, each doing a full TCP + login handshake. It parks these connections in the pool. And this is a synchronous job hence application bootup stops till the pool is filled.
- Spring Boot Hibernate `EntityManagerFactory` On top of that data source.
- Hibernate scans for entity classes and builds a metamodel. That is table column mapping. It also builds L2 cache if enabled. If ddl-auto is set to true it validates and generates the DDL
- For every repository interface we have declared which extends JPA repository spring generates a proxy implementation. And wire it to use the shared `EntityManagerFactory`.
- Now the app is ready and the connection is set idle in the pool.

### Build a DB connection:
- ORM never directly builds a connection. Everything starts at startup with HikariPool. Which calls driver manager which calls JDBC driver. And that JDBC driver creates the actual TCP connection.
- Hibernate simply asks Spring for a connection. Spring asks Hikari for a connection from the pool only when the connection pool needs to grow does Hikari actually make the call to DB. Otherwise, it reverts the connection which it already has in its pool.

### Connection pool 
- Spring Boot auto-configures Hikari CP as the data source. Hikari interns create the pool of connections as said earlier 

### Borrow connection 
- We never explicitly call `datasource.getconnection()`. Transaction manager borrows one lazily - only when the first SQL query needs to be fired. And binds it to the current thread 

### Mark the block atomic. 
- This also we never explicitly write. @Transactional annotation on service-level method takes care of this - begin transaction on method entry and commit on normal return. And rollback on runtime exception note: checked exception, don't trigger rollback by default. This is where the persistence context is created. It lives as long as the transaction lives. 

### Send statements
- We never generate an SQL statement. All we write is a `.save(entity)` method. And Hibernate generates the equivalent SQL statement. INSERT/UPDATE
- These statements aren't sent immediately. They are queued in the persistence context and then flushed (batched, ordered - first inserts, then updates, then deletes) right before the commit.

### Commit or rollback
- Again @Transactional annotation, Does this automatically for us when returning from the method. On commit flush the existing changes and run an actual COMMIT persistence context is cleared and closed. 

### Return connection to the pool
- This is also automatic. Spring takes care of returning the connection pool back to HikariCP. 


---
## Manual way

```java
// --- Step 2: pool, configured manually instead of via application.yml ---
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/quizstakes");
config.setUsername("app");
config.setPassword("secret");
config.setMaximumPoolSize(10);
config.setConnectionTimeout(30000);
HikariDataSource dataSource = new HikariDataSource(config); // pool fills here, while the new Hikari data source is being created, we also start creating the connections and add them to the pool. 

// --- Hibernate needs a SessionFactory built on top of that DataSource ---
StandardServiceRegistry registry = new StandardServiceRegistryBuilder()
        .applySetting("hibernate.connection.datasource", dataSource)
        .applySetting("hibernate.hbm2ddl.auto", "none")
        .build();

Metadata metadata = new MetadataSources(registry)
        .addAnnotatedClass(Wager.class)
        .buildMetadata();

SessionFactory sessionFactory = metadata.buildSessionFactory();

// --- Step 3: borrow (Session wraps a Connection) ---
try (Session session = sessionFactory.openSession()) {

    // --- Step 4: mark the atomic block, manually ---
    Transaction tx = session.beginTransaction();  // == setAutoCommit(false) under the hood

    try {
        // --- Step 5: statements — Hibernate generates SQL, queues in persistence context ---
        Wager wager = new Wager();
        wager.setUserId(42L);
        wager.setAmount(new BigDecimal("50.00"));
        session.persist(wager);          // not sent yet — buffered

        FundsLedger ledger = session.get(FundsLedger.class, 42L);
        ledger.debit(new BigDecimal("50.00")); // dirty-checked, not sent yet

        // --- Step 6: commit — this is where flush + physical COMMIT actually happens ---
        tx.commit();

    } catch (RuntimeException e) {
        tx.rollback();   // manual rollback path — @Transactional does this for you on unchecked ex
        throw e;
    }

} // Session.close() → connection returned to Hikari pool (step 7)
```


---
### Miscellaneous
- Default for ports for various RDBMS systems
  Postgres: 5432
  MySQL/MariaDB: 3306
  Oracle: 1521
  SQL Server: 1433

