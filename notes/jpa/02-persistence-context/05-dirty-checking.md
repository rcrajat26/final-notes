## Dirty checking
### The mechanism 
- At flush time, for every entity currently `MANAGED` in the PC, `DefaultFlushEntityEventListener` runs:
```
dirtyProperties = persister.findDirty(currentState, entry.getLoadedState(), entity, session)
```
- `currentState` is read fresh from the entity right then. `entry.getLoadedState()` is the snapshot. 
- For each mapped property, the property's `Type.isDirty(old, new, session)` decides — for basic types this typically boils down to `!Objects.equals(old, new);` for association types, it compares the FK value, not the associated object's own fields.
- If any property is dirty, the entity is marked dirty and an `EntityUpdateAction` is queued for execution.
- Crucially: this scan runs for every managed entity, every flush, regardless of whether you touched it. This is O(managed entities × mapped properties) per flush.
- `@Transactional` method loads a thousand Account rows into one PC and mutates none of them, Hibernate still walks all thousand entities' property arrays at flush to confirm nothing changed.

**Scenario 1 — plain mutation, no explicit save**
```java
Account acc = em.find(Account.class, 5L);
acc.setStatus(AccountStatus.SUSPENDED);
// no acc.save(), no em.persist(acc), nothing — just the setter
```
- At flush, `findDirty` sees `status` differs from the snapshot → `UPDATE account SET status = 'SUSPENDED' WHERE id = 5` is generated automatically.
- for a managed entity, the setter is the persistence operation; save()/persist() only matters for new entities.

**Scenario 2 — mutate then revert before flush**
```java
Account acc = em.find(Account.class, 5L);        // status = ACTIVE
acc.setStatus(AccountStatus.SUSPENDED);
acc.setStatus(AccountStatus.ACTIVE);              // back to original, before flush ever runs
```
- `findDirty` compares final current state (`ACTIVE`) against the snapshot (`ACTIVE`) — no difference.
- `dirtyProperties` comes back empty for this entity, no `EntityUpdateAction` is even queued, no SQL executes at all.
- The scan still happens (cost isn't avoided), but the result produces zero write.

**Scenario 3 — dirty checking interacting with optimistic locking**
```java
@Entity
class Account {
    @Version
    private Long version;
    ...
}
```
If `status` is found dirty in scenario 1, and `Account` has a `@Version` field, the generated `UPDATE` becomes:
```sql
UPDATE account SET status = 'SUSPENDED', version = version + 1
WHERE id = 5 AND version = <the version read at load time>
```
- Zero rows affected means someone else updated this row since you loaded it — Hibernate detects that and throws `OptimisticLockException`.

**Scenario 4 — dirty checking does NOT cover collections the same way**
```java
Client client = em.find(Client.class, 7L);
client.getAccounts().add(newAccount);   // does NOT dirty Client's own property array
```
- `findDirty` on `Client` checks `name`, `email`, `contactNumber` — none changed.
- `CollectionEntry` for `accounts` is a separate mechanism. It tracks additions/removals to the collection itself, and at flush time, it generates the necessary `INSERT`/`DELETE` statements for the join table or FK updates, but it does not mark the owning entity as dirty unless one of its own properties changed.
- which is why this never produces `UPDATE client SET ...`.

**Scenario 5 — `@DynamicUpdate`**
- By default, Hibernate generates `UPDATE` statements that include all mapped columns of the entity, even if only one property changed. This can be inefficient for wide tables.
- `@DynamicUpdate` tells Hibernate to generate `UPDATE` statements that only include the columns that were actually changed (the dirty properties). This can reduce the amount of data sent to the database and improve performance, especially for entities with many columns.

```java
@Entity
@DynamicUpdate
class Account { ... }
```

```sql
-- without @DynamicUpdate, updating just status still writes:
UPDATE account SET account_type=?, status=?, client_id=? WHERE id=?
-- with @DynamicUpdate, same scenario writes only:
UPDATE account SET status=? WHERE id=?
```

--- 
## Miscellaneous 
- **Why a detached entity cannot be dirty-checked**?: No EntityEntry, therefore no loadedState. That is precisely why merge must re-read the row to compute a diff.
- **The cost**: O(entities × properties) per flush, and there can be several flushes per transaction. 10,000 entities × 20 properties × 3 flushes = 600,000 comparisons.