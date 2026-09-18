## Life cycle inside persistence context 
### What happens inside a find flow? 
```java
Client client = em.find(Client.class, 7L);
```
- Let's say we executed the above code: `EM.find` 
- Build EntityKey(Client, 7)
- Check the PC's identity map for that key. Miss (first time).
- Persister builds and executes SELECT * FROM client WHERE id = 7
- ResultSet hydrated into an Object[] of property values
- `TwoPhaseLoad.initializeEntity`: instantiate the Client object, set its fields from the hydrated array, copy that same array into `EntityEntry.loadedState` as the snapshot.
- `EntityEntry` created with `status = MANAGED`, `lockMode = READ`, registered under `EntityKey(Client, 7)` in the PC.
- `PostLoad` lifecycle callback fires, if you have one.
- The `Client` object is now in the persistence context, and any changes made to it will be tracked. If you call `em.find(Client.class, 7L)` again within the same persistence context, it will return the same instance of `Client` without executing another SQL query, ensuring that there is only one Java object representing that database row.

### What happens inside a persist flow?
```java
Account acc = new Account(AccountType.CFT, AccountStatus.ACTIVE);
em.persist(acc);
```

- Let's say we call the above code `EM.persist` 
- acc starts life TRANSIENT — not in the PC at all.
- Hibernate resolves the id-generation strategy. Assume SEQUENCE: it calls the sequence generator now, gets back a real id, and assigns it to acc.id immediately — before any SQL INSERT runs.
- `EntityKey(Account, <new id>)` is created and registered in the PC right away, with `EntityEntry.status = MANAGED`.
- No `loadedState` snapshot is meaningfully set yet in the sense of "what's in the DB" — there's nothing in the DB to snapshot against. Hibernate just knows "this is a new row, insert it as-is."
- An `EntityInsertAction` is appended to `ActionQueue.insertions` (or `unresolvedInsertions` if its FK target — here, `client`, if set — isn't itself resolved yet).
- Nothing has hit the database. Your acc object is fully usable, mutable, managed — but no row exists yet.
- At flush (auto-triggered before commit, before a query that would need consistent state, or by explicit `session.flush()`): `ActionQueue` topologically sorts `insertions`, then executes each: builds `INSERT INTO account (...) VALUES (...)` from acc's current field values, runs it via JDBC.
- Post-execution, `EntityEntry.loadedState` is now populated to match what was just written — this becomes the baseline for any future dirty check in this same transaction, if you mutate acc again before commit.

### What happens in a combined flow of find and persist?
```java
@Transactional
public void openCftAccount(Long clientId) {
    Client client = em.find(Client.class, clientId);      // find lifecycle, steps above from em.find() flow

    Account acc = new Account(AccountType.CFT, AccountStatus.ACTIVE);
    acc.setClient(client);
    client.getAccounts().add(acc);                        // CollectionEntry initializes + marks dirty
    em.persist(acc);                                        // insert lifecycle, steps above from em.persist() flow

    client.setEmail("updated@example.com");                 // mutates a MANAGED entity's field directly
}                                                            // method ends -> flush -> commit
```

- Let's say we execute the above code 
- `client`'s dirty check finds `email` differs from its `loadedState` snapshot → `EntityUpdateAction` queued. 
- `acc`'s `EntityInsertAction` (queued back at `persist()`) executes. 
- `client.accounts`'s `CollectionEntry` is dirty (new element) — but since `Account` owns the FK, this doesn't generate its own SQL; it just confirms the cascade already handled it via `acc`'s insert.
- `ActionQueue` topologically orders these so `EntityUpdateAction` (on client, an existing row) and `EntityInsertAction` (on acc, referencing client_id) can both run safely — the FK is already satisfiable since `client.id` existed before this transaction even started.

### What happens when we have both client and account entities in a new persist? 
- unlike the earlier example, `client.id` does not exist before the transaction starts — both rows are brand new.

**Wrong approach**
```java
Client client = new Client("Rajat", "rajat@example.com", "+91...");
Account account = new Account(AccountType.CFT, AccountStatus.ACTIVE);
account.setClient(client);

em.persist(account);   // client is never persist()'d, and Client.accounts has no cascade- 
```

- In this case we get TransientPropertyValueException
- Why: `account`'s `client` field points at a Client object that has no `EntityKey`, no `EntityEntry`, and no id — it was never made `MANAGED`, and nothing cascaded a `persist()` onto it.
- When Hibernate tries to build `account`'s `INSERT`, it needs a value for the `client_id` FK column
- It looks up `client` in `entitiesByKey`/`entityEntryContext` to resolve its identifier — and finds nothing, because client was never registered.
- so Hibernate refuses outright rather than insert a broken/null FK silently

### The right approach — two correct variants
**Variant A: explicit, persist the parent first**

```java
Client client = new Client("Rajat", "rajat@example.com", "+91...");
em.persist(client);              // client -> MANAGED, id assigned immediately (SEQUENCE)

Account account = new Account(AccountType.CFT, AccountStatus.ACTIVE);
account.setClient(client);
em.persist(account);             // client is now resolvable -> FK can be built
```

- Because your `Client.id` uses `GenerationType.SEQUENCE`, the first `persist(client)` call gets a real id back from the sequence immediately — client is registered in `entitiesByKey`/`entityEntryContext` right then, well before flush
- By the time `account.setClient(client)` happens and `persist(account)` runs, client is fully resolvable, and account's EntityInsertAction can be built with a real `client_id` value straight away — no waiting, no unresolvedInsertions needed at all here, since the dependency was already satisfied at persist-time, not just at flush-time.

**Variant B: idiomatic, rely on cascade**

```java
@OneToMany(mappedBy = "client", cascade = CascadeType.PERSIST, orphanRemoval = true)
private List<Account> accounts = new ArrayList<>();
```

```java
Client client = new Client("Rajat", "rajat@example.com", "+91...");
Account account = new Account(AccountType.CFT, AccountStatus.ACTIVE);
account.setClient(client);
client.getAccounts().add(account);

em.persist(client);   // cascades PERSIST down to `account` automatically
```

- Here you only call `persist()` once, on the parent. 
- Because `Client.accounts` has `cascade = CascadeType.PERSIST`, Hibernate's cascade engine walks `client.getAccounts()` during this single `persist()` call and invokes the equivalent of persist(account) on each element itself — so account ends up registered in the PC in the same operation, without you calling `persist()` on it directly. 
- Functionally identical outcome to Variant A, less code, and it's the shape you'd actually want for Client/Account since a client onboarding a new account is naturally "one aggregate root operation."