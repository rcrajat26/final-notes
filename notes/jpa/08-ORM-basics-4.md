# ORM
## ORM components 

### EntityManagerFactory 
- An entity manager factory is a heavyweight object. Created once for a DB, it is expensive to construct, it is immutable and thread-safe: 
  - It parses the persistence.xml or the Spring Boot autoconfiguration.
  - Builds metamodel by scanning all the entity classes.
  - Creates `EntityPersister` For each entity class which holds all the related SQL statements
  - It holds the Dialect — a ~40-method class that knows given database's LIMIT syntax, sequence syntax, lock hints, temp-table strategy, type mappings. It diffs for each database type: Oracle, Postgres, MySQL.
  - It holds Proxy subclasses created for lazy loading per entity class
  - Sets up connection pool and builds L2 cache if enabled. 
- In a Spring Boot class we almost never create or touch it directly. Spring creates and manages an entity manager factory.
- It lives as long as the application context of Spring Boot lives. 
- A close method, `.close()`, is called on EMF during the application shutdown. This is also taken care of by the Spring itself when the context closes. Calling it midway through any operation would fail every persistence operation in the application. It essentially - shuts the pool, evicts L2 regions, stops internal executors, if ddl-auto is set to create drop, runs DDL ops to drop schema.
- Startup cost breakdown: annotation scan → metadata binding → persister construction → proxy class generation → SQL string pre-generation → query-plan warm-up → ddl-auto execution.

### EntityManager
- An entity manager is a lightweight object. It is not thread-safe, and it is cheap to construct. It is created per transaction or per request. It represents a single persistence context, which is a first-level cache of entity objects. 
  - It tracks the state of each entity (new, managed, detached, removed) and synchronizes changes to the database when the transaction commits or an explicit flush. 
  - We get typically one for each unit of work 
- The `.close()` method call on this closes the persistence context and detaches all the managed entities.


### Transaction 
- A transaction is a unit of atomicity wrapped around entity manager operations - Boundary within which all changes, commit or everything rolls back.
- JPA-native: entityManager.getTransaction().begin() / .commit() / .rollback() — used in plain JPA 
- Spring-managed: @Transactional — Spring wraps your method in a transaction using the PlatformTransactionManager, calling begin/commit/rollback around your method body, and binding the EntityManager's persistence context to that transaction's scope
- A transaction doesn't have a "close" method itself — it's terminated via commit() or rollback(), not close()