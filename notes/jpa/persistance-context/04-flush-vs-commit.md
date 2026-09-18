## Flush versus commit in detail
### Flush

- Hibernate is deliberately lazy about generating SQL, for a real reason: batching and ordering. 
- If every setter call fired its own UPDATE immediately, you'd lose the ability to batch multiple changes into fewer round trips, and you'd lose the topological ordering we discussed for inserts (parent-before-child) — because at the moment you call one setter, Hibernate doesn't yet know what else you're about to do in this transaction.
- So Hibernate accumulates: the ActionQueue fills up with pending EntityInsertAction/EntityUpdateAction/etc. as you make changes, and nothing is actually sent to the database until some triggering event says "okay, now generate and execute the SQL for everything pending." That triggering event is flush.
- After a flush, SQL has been executed against the database, but the database transaction is still open.
- Nothing has been made permanent. If the transaction rolled back right after this flush, every one of those statements would be undone

```
Flush = the process of synchronizing the persistence context's in-memory state with the database, by generating and executing
the SQL for everything in the ActionQueue, plus running dirty-checking first to populate that queue with any pending updates.
```

**What flush does**:
1. Dirty-check every managed entity — for each `EntityKey → EntityEntry` in the PC, compare current field values against the loaded snapshot (`EntityEntry.loadedState`). Any differences get an `EntityUpdateAction` appended to the queue right now, at flush time — not before. 
2. Cascade — process any cascade-eligible operations that haven't been resolved yet (e.g. cascading dirty child entities if applicable). 
3. Sort and execute the `ActionQueue`: topologically sort insertions, then execute inserts, updates, deletes, and collection actions in dependency-safe order — each one becomes an actual JDBC statement sent to the database, right now. 
4. Refresh snapshots — for each entity just updated/inserted, `EntityEntry.loadedState` is re-synced to the just-written values, so the next flush (if any, before commit) has a correct new baseline.

### Commit
```
Commit = ending the database transaction successfully, making every statement executed within it — flushed or not — permanent and visible to others, and releasing any locks held.
```
- At the Spring/JPA layer, commit is driven by the transaction manager, triggered when a `@Transactional` method returns normally.
- Before issuing the actual `COMMIT` to the database, Spring/Hibernate does one more thing: it flushes first, if anything is still pending

### Why a separate flush when commit already triggers it?
- Because flush is a separate operation, you can call it explicitly in the middle of a transaction if you want to ensure that the database sees your changes before the transaction ends. This can be useful for:
    - Validating constraints that are enforced at the database level (e.g., unique constraints) before committing.
    - Triggering database-generated values (like auto-incremented IDs) to be available in your application code before the transaction ends.
    - Ensuring that any side effects of triggers or stored procedures are executed and visible within the same transaction.

**Trigger 1 — explicit em.flush() / session.flush()**: 
- You call it yourself. 
- Rare in normal application code, but genuinely useful when you need the generated id of a newly-persisted entity to use it in a subsequent operation within the same transaction, before that transaction ends.


**Trigger 2 — auto-flush before a query, when the default FlushModeType.AUTO is in effect**:
- If you mutate a managed entity, then run a query that could be affected by that mutation, Hibernate flushes first to keep the query's results consistent with your in-memory changes:
```java
@Transactional
public void demonstrateAutoFlush() {
    Account account = em.find(Account.class, 5L);
    account.setStatus(AccountStatus.SUSPENDED);

    // no explicit flush call here, but:
    List<Account> suspended = em.createQuery(
        "SELECT a FROM Account a WHERE a.status = :s", Account.class)
        .setParameter("s", AccountStatus.SUSPENDED)
        .getResultList();

    // Hibernate detects: a query is about to run against the `account` table,
    // and there's a pending dirty change to an Account entity that could affect
    // this query's result set -> auto-flushes BEFORE executing the SELECT.
    // Without the auto-flush, `suspended` would miss the row you just changed
    // in memory, because the DB wouldn't reflect it yet.

    // suspended now correctly includes account, id=5
} 
```

**Trigger 3 — FlushModeType.COMMIT**
- You can tell Hibernate to skip the auto-flush-before-query behavior and only ever flush at commit time
```java
em.setFlushMode(FlushModeType.COMMIT); 
```
- With this set, the `demonstrateAutoFlush()` example above would run its `SELECT` without flushing first — suspended would not include the in-memory-only change, because the database genuinely doesn't have it yet.
- This mode exists for read-heavy transactions where you want tight control over exactly when writes go out, but it's a footgun if you forget you set it and then write a query expecting to see your own uncommitted-in-memory changes.

| Mode | Behaviour |
|---|---|
| `FlushModeType.AUTO` (default) | flush at commit, **and** before a JPQL/HQL/Criteria query whose result could be affected by pending changes |
| `FlushModeType.COMMIT` | flush only at commit; queries may then read stale data |
| `FlushMode.MANUAL` (Hibernate-only) | **never** flush automatically, not even at commit |
| `FlushMode.ALWAYS` (Hibernate-only) | flush before every query regardless of query space — the correctness sledgehammer |

**Note: Does flush add to the ActionQueue and almost immediately execute the same ActionQueue?"**
```
- For updates and collection dirtiness: yes, exactly that — flush is the only moment those actions come into existence at all (there's no earlier 
API call that would trigger their creation, since a plain setter call gives Hibernate no signal), and they get executed moments later within 
that same flush invocation.
- For inserts and deletes: no — those were already sitting in the queue since the earlier persist()/remove() call. 
Flush's role for those is purely execution (plus sorting); it adds nothing new for them.
```

---

## Miscellaneous 

### readOnly = true and MANUAL

`@Transactional(readOnly = true)` sets Hibernate's flush mode to **`MANUAL`**, so:

```
normal:    commit() → [dirty-check 10,000 entities] → COMMIT
readOnly:  commit() → COMMIT
```

That is what "straight to commit" means — the entire dirty-check pass is skipped. It also flags
the JDBC connection read-only (a hint some drivers use to route to a replica).

> `[TRAP]` Under `MANUAL`, a `save()` you accidentally call in a `readOnly` method **does
> nothing**. No error, no SQL, no row. The most confusing silent failure in the whole topic.

> `[TRAP]` `readOnly = true` does **not** make entities immutable and does not prevent writes.
> An explicit `flush()`, a `@Modifying` query, or a nested writable transaction still writes.
> Some databases reject writes on a read-only connection; most do not.

