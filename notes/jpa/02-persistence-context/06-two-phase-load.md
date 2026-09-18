## Two-phase load 
### The problem 
```java
Client client = session.get(Client.class, 1L);
```
- Let's say we run the above Java statement 
- Suppose `Client` has `List<Account> accounts` and each `Account` has `Client client` (bidirectional). When Hibernate loads the `Client` row, it eventually needs to resolve the `accounts` association. 
- But resolving an `Account` means potentially resolving its `client` reference back to... the same `Client` you're still in the middle of loading.
- If you tried to build the `Client` object fully in one shot — read columns, immediately turn foreign keys into resolved object references, immediately populate collections — you'd recurse forever, or at minimum you'd need the fully-formed `Client` to exist before it exists.

### The solution: two-phase load 
- The fix: split loading into two phases, with a registration step in between.

**Phase 1: Hydration**
- When Hibernate executes the SQL and gets a ResultSet row back, it does not immediately build your Client entity. Instead, EntityPersister.hydrate() reads the raw column values off the JDBC ResultSet into a plain Object[] — the "hydrated state."

For the below row:
```java
id=1, name='Rajat', email='r@x.com', contact_number='999...'
```

hydrated state looks like:
```java
Object[] hydratedState = { "Rajat", "r@x.com", "999..." };
```

- This is just a plain array of values, not yet a Client object. 
- It also does not yet resolve any foreign keys into actual object references. The collections are handled separately, lazily.

**The registration step**
- Immediately after hydrating, Hibernate does something important: it creates the (still mostly empty) entity instance (via reflection) and registers it in the PersistenceContext under key (Client.class, 1L) before populating its fields, with a status of LOADING.
```java
// conceptually, inside TwoPhaseLoad.initializeEntity()
Object entity = persister.instantiate(id, session);      // empty shell
persistenceContext.addEntity(entityKey, entity);           // registered NOW
persistenceContext.addEntry(entity, Status.LOADING, ...);  // marked LOADING 
```

**Phase 2: Resolution**
Now TwoPhaseLoad takes the hydrated Object[] and actually resolves it into final property values on the entity:
- Simple types (name, email, contactNumber) are set as-is.
- To-one associations get resolved into either the real object (if already in context/cache) or a Byte Buddy proxy (if lazy and not yet loaded) — session.internalLoad() is what mints that proxy.
- The entity's status moves from LOADING → MANAGED
- PostLoadEvent fires (this is when @PostLoad callbacks run — notice they don't run until resolution finishes).

```java
persister.setPropertyValues(entity, resolvedValues);
persistenceContext.getEntry(entity).setStatus(Status.MANAGED);
```

- Only after this does session.get(Client.class, 1L) actually hand you back something usable.

### A concrete example 
```java
Client client = session.get(Client.class, 1L);
// client.getAccounts() -> lazy PersistentBag, not yet initialized
```

1. SQL for client row runs → hydrate → Object[]{"Rajat","r@x.com","999..."}
2. Empty Client shell instantiated, registered in PersistenceContext as LOADING for key (Client, 1L)
3. Resolution: scalar fields set. The accounts collection field is set to an uninitialized PersistentBag wrapper — not queried yet at all. This is a separate, lazy mechanism layered on top of two-phase load. 
4. Status flips to MANAGED. PostLoad fires. You get your client back.

**next,**
```java
client.getAccounts().size(); // triggers collection initialization
```

- This fires a second query (`SELECT * FROM account WHERE client_id = 1`), and each `Account` row goes through its own two-phase load
- hydrate `Object[]{status, type}` + the raw `client_id` FK, register the `Account` shell as `LOADING`, then resolve: the `client_id` FK gets resolved via `session.internalLoad(Client.class, 1L)`
- Here's the payoff: `internalLoad` checks the `PersistenceContext` first, finds `Client #1` already sitting there as `MANAGED`, and just wires up that same reference — no new query, no recursion back into loading `Client` again.
- If instead `Account` were loaded first in some cascading fetch and referenced a `Client` not yet in the context, `internalLoad` would return a `proxy` rather than recursing into a full load right there — deferring the actual `SELECT` until the proxy is touched.

### Advantages 
- It's the reason self-referencing entities (`Employee.manager -> Employee`) and bidirectional associations don't blow the stack.
- It's why `@PostLoad` sees fully-resolved associations, not raw FK values.
- It explains why a `LazyInitializationException` happens after session close specifically on the collection/proxy resolution step — that step is deferred, separate from the two-phase load of the owning entity itself.
- It's the reason the identity map guarantee holds ("within one session, the same row = the same Java object") — the registration happens before resolution, so nothing can slip through and create a duplicate.

---

## Miscellaneous 
### How object[] ensures the field map guarantee
- At bootstrap, Hibernate walks `Client`'s mapped properties and builds an ordered list — say, alphabetical-ish by default but really "whatever order the metamodel binding produced," typically: `[contactNumber, email, name]` (identifier is excluded — it's handled separately). This becomes `EntityPersister.getPropertyNames()`, and it is fixed for the lifetime of that persister — same JVM run, same order, every time.
- The SQL itself is generated from this exact same list — the SELECT statement's column order in EntityPersister's cached SQL string matches the property order 1:1. So the ResultSet you read from is already in the right order because Hibernate wrote the query that produced it.
- hydrate() just does for (i = 0; i < propertyNames.length; i++) state[i] = types[i].nullSafeGet(resultSet, ...) — reading in that fixed order into the array.
- When it comes time to resolve, setPropertyValues() does the same loop in the same order, so state[i] is guaranteed to correspond to propertyNames[i] — no map lookups, no reflection, no guessing. This is how Hibernate guarantees that the right value goes into the right field every time, even across multiple flushes and loads.

So it's positional, yes — but the position is derived from a single canonical ordering computed once at startup and reused for both SQL generation and array read/write.