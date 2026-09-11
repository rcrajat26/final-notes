## ORM
- ORM is an object state management system, its actual job is to:
  - Track the state of an entity object (managed, transient, detached, removed)
  - Detect changes to that state. 
  - Decide when and how to synchronize those states to the database 
  - Produce SQL as a side effect of this. 
- So what happens is: 
  - your code mutates objects → 
  - persistence context notices → 
  - flush triggers synchronization → 
  - dialect-aware SQL renderer produces the actual statement.
- Hence we can't directly decide how the SQL statement is generated. 
- For ex:
```java
@Transactional
public void placeBet(Long userId, BigDecimal amount) {
    FundsLedger ledger = ledgerRepo.findById(userId).orElseThrow();
    ledger.debit(amount);  // just a setter, no SQL here
    // ... more business logic, no repository.save() call anywhere ...
} 
```
- Nobody called save(). Nobody wrote an UPDATE statement. But when the transaction commits, Hibernate flushes the persistence context, notices the FundsLedger entity's balance field changed since it was loaded, and issues an UPDATE.
- You never asked for SQL to be generated — you asked for object state to change, and SQL happened because Hibernate is watching.

---
### What ORM does?
- Mapping metadata: using annotations like `@Entity`, `@OneToMany`, `@Column`, Which establishes connectivity between the fields of objects and the columns of the table along with the relationships other metadata.
- Identity management: It does identity management within a single session by not giving two different results for the same primary key fetch.
- State tracking and lifecycle management: it manages the state of an entity object. And helps transition between them.
- Transactional rate behind: it decides when to make the actual call to the database rather than doing it per query or per update basis. 
- Query translation: it translates the JPQL or HQL queries to SQL queries.
- Relationship loading strategies: Deciding how to fetch associated objects using FetchType EAGER/LAZY, @ManyToOne and @OneToOne default to EAGER, while @OneToMany and @ManyToMany default to LAZY 

---
## Miscellaneous
### The proxy object in lazy loading
- When we are trying to lazy load a certain object there must be something in its place. Which exactly mimics the class that we want to return. Here in other words, we need a proxy for the given class. 
```java
Bet bet = betRepo.findById(betId).orElseThrow();
FundsLedger ledger = bet.getFundsLedger(); 
```
- In the above code we don't know when `ledger.getbalance()` will be called. Hence we'd like to lazy load this. Instead of returning the same object, we return the object which is a proxy of this object. Note we can't return `null` because doing an operation on `null` will give a null pointer exception. And also we can't give a static object each time because each persistence context would need a different copy, reference to entity manager so that it makes an actual SQL call when the object is requested. It also needs to overload the getter methods to accommodate logic check to verify if the object already exists. If it doesn't, it makes a SQL call. If it exists it simply gives whatever object is needed. 
- At startup Hibernate builds a proxy class for this particular funds ledger class. `FundsLedger$HibernateProxy$abc123 extends FundsLedger`.
- This proxy class will hold only the ID of the funds ledger. And a reference to the current hibernate session
- It overrides every getter/setter. To check, has the real data already been loaded? 
- If not loaded make a SQL call to the database. Populate internal target instance this target instance is the actual instance of funds ledger inside the proxy funds ledger. Then delegate the call to it. 
- If loaded already then directly delegate the call. 

### Dialect mapping
- SQL is not a standard language. Each database vendor has its own dialect. Hence Hibernate needs to know which dialect to use for generating SQL statements.
- Hibernate defines an abstract class org.hibernate.dialect.Dialect, and ships one concrete subclass per supported database — PostgreSQLDialect, OracleDialect, MySQLDialect, SQLServerDialect, H2Dialect, and so on.
- We configure exactly one of them in the properties

```yaml
spring:
  jpa:
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
```

### Note, choose right:
ORM for transactional CRUD on an object graph. JDBC / jOOQ / native SQL for reporting, bulk updates, and anything where the query shape matters more than the object shape. Mixing is normal and correct, not a failure. Both share the same transaction and connection under JpaTransactionManager.