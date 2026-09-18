### Background
In plain Java, equals/hashCode are supposed to be simple: same logical object → same result, consistently, forever. Hibernate breaks the assumptions that make this easy, in three specific ways:
1. An entity's ID doesn't exist yet at some point in its life. A new Customer() has id = null until you save()/flush() it. But you might put that object into a HashSet before it's saved.
2. Hibernate hands you proxies, not always the class you expect. session.load() or a lazy @ManyToOne gives you a CGLIB/ByteBuddy subclass, not Customer.class — so getClass() == getClass() checks silently fail even for the "same" entity.
3. The same database row can be represented by different Java object instances across different persistence contexts (different EntityManagers, different requests). Java's default equals (reference equality) says these are different objects. They're the same row.

```
So you cannot just leave equals/hashCode unimplemented (default identity semantics breaks anything that treats "same row" as "same object" across sessions), and the three "obvious" fixes each have a landmine.
```

## equals/hashCode on entities is a minefield
### Negative scenario A — default identity equals across two PCs
```java
Client c1 = requestA_entityManager.find(Client.class, 7L);
Client c2 = requestB_entityManager.find(Client.class, 7L);
c1.equals(c2); // false — different Java objects, default Object.equals is reference equality
```
- Both represent the same database row, but across two separate HTTP requests (two separate persistence contexts)
- the identity-map guarantee doesn't span PCs
- This is a common source of confusion and bugs. If you rely on `equals` to compare entities across different persistence contexts, you may get unexpected results.

### Negative scenario B — id-based hashCode, id assigned after construction
```java
@Override
public int hashCode() { return id.hashCode(); }  // NPE risk aside, id is null before persist
```

```java
Set<Account> pending = new HashSet<>();
Account acc = new Account(...);        // id == null, hashCode = some value (or NPE)
pending.add(acc);                      // bucketed using that hashCode
em.persist(acc);                       // id now assigned — hashCode value has changed!
pending.contains(acc);                 // may return false — wrong bucket now
```
- The object got filed into the HashSet under one hash bucket, then its effective hash changed after mutation — a hashCode contract violation. 
- The object isn't lost from memory, but it's unreachable via the Set's lookup because the bucket it's actually sitting in no longer matches where hashCode() would tell you to look.

### Negative scenario C — proxies and getClass()
```java
@Override
public boolean equals(Object o) {
    if (getClass() != o.getClass()) return false;   // fails for proxies
    ...
}
```

```java
Account real = new Account();
Account proxy = em.getReference(Account.class, 5L);  // returns a Byte Buddy subclass proxy
real.equals(proxy); // false — proxy.getClass() is Account$HibernateProxy$..., not Account.class
```
- Correct version uses instanceof (or Hibernate.getClass(o) to unwrap the proxy) rather than a strict class-equality check.

## Solutions 
### Option A — Business/natural key
- preferred where one genuinely exists — e.g., a unique email, an external reference number that's set at construction and never changes

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Customer other)) return false;
    return Objects.equals(email, other.email); // stable from creation, never null, never changes
}

@Override
public int hashCode() {
    return Objects.hash(email);
}
```

### Option B — Defensive ID-based equals
```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Customer other)) return false;
    Class<?> thisClass = Hibernate.getClass(this);   // unwrap proxy
    Class<?> otherClass = Hibernate.getClass(other);
    if (!thisClass.equals(otherClass)) return false;
    return id != null && id.equals(other.id);        // null id -> never equal to anything, even itself-by-value
}

@Override
public int hashCode() {
    return getClass().hashCode(); // CONSTANT — doesn't depend on id at all
}
```

- The trick: hashCode() returns a constant (or is based only on the class), never on the mutable/assigned id. That means every instance of Customer lands in the same hash bucket
- but it makes hashCode() stable across the entity's entire lifecycle (transient → persistent) which is the actual contract requirement
