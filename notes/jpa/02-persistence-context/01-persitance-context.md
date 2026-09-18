# Persistence context 
## Persistent context definition: What is persistent context? 

- Persistence context is the core of Hibernate 
- It's a unit of work + an identity map, scoped a single Hibernate session
- It's a set of in-memory data structure that Hibernate keeps

To define:
```
A persistence context is an in-memory cache of managed entities, scoped to one Session/EntityManager, that guarantees at most
one Java object per database row, tracks each entity's state (identity map + EntityEntry), remembers what it looked like when 
loaded (the snapshot), and queues up the SQL needed to reconcile any changes (the ActionQueue) — deferring that SQL until flush.
```

It's a set of in-memory data structures Hibernate keeps per-session:
### EntityKey: 
is a composite of (entity name, identifier value)

### EntityEntry: 
is the metadata Hibernate attaches to each managed instance: 
- its Status (MANAGED, READ_ONLY, DELETED, GONE), 
- its LockMode, a reference to the entity's persister, 
- and the loaded state snapshot

ex:
```
Client client = em.find(Client.class, 7L);
```
After this call, the persistence context internally has:
```
EntityKey(Client, 7) -> EntityEntry {
    status: MANAGED,
    lockMode: READ,
    loadedState: ["Rajat", "rajat@example.com", "+91..."],
    entity: <reference to the client object>
}
```
Note: Call em.find(Client.class, 7L) again in the same transaction, and Hibernate checks this map before touching the database. Since EntityKey(Client, 7) is already present, it returns the same object reference — no SQL.

### CollectionEntry map: 
Every persistent collection field (Client.accounts here) is, at runtime, not a plain ArrayList — Hibernate wraps it in a PersistentBag (for a List without an explicit order column), or PersistentSet/PersistentList/PersistentMap for other collection types. This wrapper is what lets Hibernate intercept add(), remove(), iteration, etc.

The CollectionEntry, keyed by the collection's role (a string like "com.quizstakes.Client.accounts" combined with the owner's id), tracks:
- loadedKey — the owner's identifier this collection belongs to
- A snapshot of the collection's own state (which element ids/values were present when loaded) — separate from the owner entity's property snapshot
- initialized — whether the lazy collection has actually been fetched from the DB yet, or is still an uninitialized proxy
- dirty — whether adds/removes have happened since load
- reached — used during cascade processing to avoid infinite loops on bidirectional graphs

ex:
```java
Client client = em.find(Client.class, 7L);          // accounts collection NOT yet loaded (LAZY)
client.getAccounts().add(newAccount);                // triggers initialization:
                                                       //   1. SELECT * FROM account WHERE client_id = 7
                                                       //   2. CollectionEntry.snapshot = [existing account ids]
                                                       //   3. PersistentBag now wraps a real ArrayList with those elements
                                                       //   4. add() appends newAccount, marks CollectionEntry.dirty = true
```

### ActionQueue
- a set of ordered lists of pending actions like: EntityInsertAction, EntityUpdateAction, EntityDeleteAction, CollectionAction — these are queued, not executed immediately, until flush time.
- `ActionQueue` is a per-session structure holding several separate typed lists of pending write actions
- Including distinct lists for entity inserts, entity updates, entity deletes, collection creations, collection updates, collection removals, and queued extra-lazy collection ops, plus a holding area for inserts whose FK dependencies aren't resolvable yet
- Actions are appended to these lists as you call `persist()`/mutate managed entities/mutate collections, but no SQL is generated until flush, at which point the queue topologically sorts the insert list (so parent rows are inserted before dependent child rows regardless of call order) and executes each list in dependency-safe order.
- One exception breaks the "queued until flush" rule entirely: `GenerationType.IDENTITY` inserts can't be queued at all, because the entity's id isn't known until the INSERT actually runs — those execute synchronously the moment persist() is called, bypassing the queue and losing batching.

| List                   | Holds                                                                    | Example trigger                                                                                                   |
| ---------------------- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| `insertions`           | `EntityInsertAction` (ID known upfront — sequence/table generator)       | New `Account` with `GenerationType.SEQUENCE`                                                                      |
| `unresolvedInsertions` | Inserts whose FK dependency isn't resolved yet (parent not yet inserted) | New `Client` + new `Account` both transient in the same flush                                                     |
| — *(identity case)*    | `EntityIdentityInsertAction` — executes immediately, not queued          | `GenerationType.IDENTITY` — ID is only known after the `INSERT` runs, so it can't wait in a queue and can't batch |
| `updates`              | `EntityUpdateAction`                                                     | `account.setStatus(SUSPENDED)` on a managed entity                                                                |
| `deletions`            | `EntityDeleteAction`, `OrphanRemovalAction`                              | Removing an `Account` from `client.getAccounts()` when `orphanRemoval=true`                                       |
| `collectionCreations`  | `CollectionRecreateAction`                                               | First-time persist of a whole collection                                                                          |
| `collectionUpdates`    | `CollectionUpdateAction`                                                 | Element order change, join-table row changes                                                                      |
| `collectionRemovals`   | `CollectionRemoveAction`                                                 | Collection cleared or replaced wholesale                                                                          |
| `collectionQueuedOps`  | `QueuedOperationCollectionAction`                                        | Extra-lazy collections where individual add/remove operations are deferred                                        |

---

## Miscellaneous 
### Where are "the two copies," per loaded object
- For every entity Hibernate loads, there are genuinely two separate physical copies of its property values sitting in memory simultaneously:
- Copy 1 — the live entity object itself. This is the actual Client instance your code holds a reference to — client.name, client.email, client.contactNumber as real Java fields, accessible via getters, mutable by setters. This is what your business logic reads and writes.
- Copy 2 — the snapshot, EntityEntry.loadedState. This is a separate Object[] array, not a reference into the Client object's fields — it's an independent extraction of the same values, taken at load time via the persister's property-extraction machinery (persister.getPropertyValues(entity)), stored inside the EntityEntry.
- Right after load, both copies hold identical values — but they are not the same memory. Copy 1 is reachable through client.name; Copy 2 is reachable only through the EntityEntry, which your code never touches directly.

```java
Client client = em.find(Client.class, 7L);

// Copy 1 — lives inside the object graph:
client.name == "Rajat"

// Copy 2 — lives inside the PC's bookkeeping, decoupled from `client`:
entityEntryContext.getEntry(client).loadedState == ["Rajat", "rajat@example.com", "+91..."]
```

```markdown
The two rules this number produces
- Never findAll() on a large table.
- In a batch loop, flush() then clear() every ~50 rows, or you will OOM.
```


### Lock mode states

Hibernate's LockMode enum (JPA's LockModeType maps onto it) — these live on the EntityEntry:

| LockMode                      | When it's set                                                                  | What it means                                                                                                                                                                   |
| ----------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NONE`                        | Entity is transient, or you explicitly bypass locking                          | No lock held at all                                                                                                                                                             |
| `READ`                        | Default after a plain `find()` / query load within a transaction               | Implicit — just means "this entity was read in this transaction," not a DB-level lock                                                                                           |
| `WRITE`                       | Automatically upgraded when an entity is inserted or updated                   | Also implicit; no additional SQL, just bookkeeping                                                                                                                              |
| `OPTIMISTIC`                  | Explicitly requested via `em.find(Account.class, id, LockModeType.OPTIMISTIC)` | At flush/commit, Hibernate re-checks the `@Version` column hasn't changed since load — no row lock, just a version comparison                                                   |
| `OPTIMISTIC_FORCE_INCREMENT`  | Explicit                                                                       | Same check, plus forces a version bump even if no columns changed — useful when a logical change (e.g. a child collection changed) should still invalidate the parent's version |
| `PESSIMISTIC_READ`            | Explicit                                                                       | Issues `SELECT ... FOR SHARE` (or equivalent) — blocks other writers, not other readers                                                                                         |
| `PESSIMISTIC_WRITE`           | Explicit                                                                       | Issues `SELECT ... FOR UPDATE` — blocks all other lockers until commit                                                                                                          |
| `PESSIMISTIC_FORCE_INCREMENT` | Explicit                                                                       | Pessimistic lock + forced version increment                                                                                                                                     |


### Where the loaded state snapshot lives

It's an Object[] stored directly on the EntityEntry, sized and ordered according to the entity persister's property span (the same ordering used to generate SQL column lists). It's populated at two points:

On load — right after the JDBC ResultSet is hydrated into property values, TwoPhaseLoad.initializeEntity copies that hydrated state into EntityEntry.loadedState before the entity is even handed back to your code.
On flush — after an EntityUpdateAction executes its UPDATE, the snapshot is refreshed to match the just-written values, so the next flush's dirty check has a fresh baseline.

For mutable value types (a Date, a byte[], a custom composite Type), Hibernate takes a deep copy into the snapshot rather than holding the same reference — otherwise if you mutated the object in place (e.g. date.setTime(...) on a java.util.Date field), the "current" state and "snapshot" would always look identical even though the DB value is stale, and dirty checking would silently miss the change. This is part of why immutable value objects (records, or at least effectively-immutable classes) are the safe default for entity fields.

For your Account.status field (a plain enum, immutable by nature — you always replace the reference via the setter, never mutate an enum instance), this deep-copy concern doesn't apply; simple immutable types are copied by reference safely.

Scope: the snapshot lives only in memory, only for the lifetime of the persistence context. Close the EntityManager/end the transaction, and it's gone — which is exactly why a detached entity has no dirty checking: there's no snapshot left to diff against.


