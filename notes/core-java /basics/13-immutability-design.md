# Immutability Design
## What immutability actually guarantees
- An immutable object guarantees that its state cannot change after construction. This means that once an instance is created, it will always represent the same value.
- Immutability is a design choice that can lead to simpler and more predictable code, especially in concurrent programming, as immutable objects can be shared freely between threads without synchronization.
- Immutability does not mean that the object cannot be referenced by multiple variables or that it cannot be used in different contexts. It simply means that the object's state cannot be altered after it has been constructed.

```java
final class Account {
    private final String accountNumber;
    private final AccountType type;
    private final List<String> transactionIds;

    Account(String accountNumber, AccountType type, List<String> transactionIds) {
        this.accountNumber = accountNumber;
        this.type = type;
        this.transactionIds = List.copyOf(transactionIds);   // defensive copy — see below
    }

    String getAccountNumber() { return accountNumber; }
    AccountType getType() { return type; }
    List<String> getTransactionIds() { return transactionIds; }

    // no setters at all
}
```

Five conditions, all required together:
1. **Class itself is final** — prevents a subclass from adding mutable state or overriding a method to break the guarantee from outside the class's own control.
2. **All fields are private final** — final stops reassignment; private stops any external code from reaching in and assigning directly, bypassing the constructor.
3. **No setter methods** — the only place any field is ever assigned is inside the constructor.
4. **Defensive copies for mutable field types** — if a field is a reference to a mutable object (a List, an array, a Date-like type), merely making the reference final doesn't stop the referenced object's contents from changing. The constructor must copy incoming mutable data, and any getter returning such a field must return a copy or an unmodifiable wrapper — never the live internal reference.
5. **No method leaks a live reference to internal mutable state** — same idea as point 4, stated generally: nothing the class exposes (constructors, getters, any method) may hand out something that lets external code reach in and mutate the object's actual internal data.

### Why defensive copying matters — the exact bug it prevents
```java
List<String> ids = new ArrayList<>(List.of("TX1", "TX2"));
Account acc = new Account("ACC123", AccountType.IGSTK, ids);

ids.add("TX3");                          // mutating the ORIGINAL list the caller still holds
System.out.println(acc.getTransactionIds().size());   // if no defensive copy: 3 — "immutable" object just changed!
```

- Without the `List.copyOf(...)` in the constructor, `Account` would simply be storing the same list reference the caller passed in. 
- The caller mutating their own copy afterward silently mutates the "immutable" object too — because there was never actually a separate copy; both variables point at one shared list on the heap. 
- The constructor's defensive copy breaks that shared reference, so acc's internal list is now a genuinely separate object that the caller has no way to reach.

The same risk exists on the getter side, independently:
```java
List<String> getTransactionIds() { return transactionIds; }   // returns the LIVE internal list directly

Account acc = new Account(...);
acc.getTransactionIds().add("HACKED");   // mutates the object's real internal state from outside!
```

- Even with a correct defensive copy in the constructor, returning the live field reference from a getter re-opens the exact same hole from the other direction. 
- `List.copyOf(...)` conveniently solves both problems at once here because it returns an unmodifiable list — so even the object returned by the getter throws `UnsupportedOperationException` on any mutation attempt, rather than silently succeeding and changing the internal state.

### "Wither" methods — how immutable objects support "changes"
Since nothing can be mutated in place, an immutable class that needs to represent "an updated version" does so by returning a brand-new object with the changed field, leaving the original completely untouched:
```java
final class Account {
    private final String accountNumber;
    private final AccountStatus status;

    Account(String accountNumber, AccountStatus status) {
        this.accountNumber = accountNumber;
        this.status = status;
    }

    Account withStatus(AccountStatus newStatus) {
        return new Account(this.accountNumber, newStatus);   // new object, old one unaffected
    }
}

Account original = new Account("ACC123", AccountStatus.PENDING);
Account activated = original.withStatus(AccountStatus.ACTIVE);
// original.status is still PENDING — untouched
// activated is a distinct object with the new status
```
This "wither" pattern is the standard idiom for evolving immutable data — the caller decides whether to keep the old reference, the new one, or both; nothing is ever silently changed underneath anyone.

### Why immutability is treated as a design goal, not just a rule for one class
- **Thread safety for free** — an object with no mutable state has nothing for concurrent threads to race on; it can be shared across threads with zero synchronization. 
- **Safe as a hash key** — an object's hash code (and equality) can never go stale after being placed into a hash-based structure, since nothing about it can change post-construction. 
- **Easier reasoning** — any method that receives an immutable object as a parameter can be certain it won't be changed by some other code holding the same reference elsewhere — eliminates an entire category of "who else has a reference to this, and did they change it on me" bugs. 
- **Safe to share without defensive copying yourself** — if Account is genuinely immutable, handing a reference to it to another part of the system needs no copying at all; the object's own immutability already protects it.

### The tradeoff, stated plainly
- Every "modification" of an immutable object allocates a new object. 
- For a Client with a large accounts list, calling withAddedAccount(...) repeatedly in a loop means allocating a new Client (and, depending on design, possibly copying the account list again) on every call — a real cost compared to in-place mutation. 
- This is a legitimate design tradeoff, not a flaw to "fix" — immutability trades allocation cost for correctness guarantees, and the right choice depends on how often the object changes versus how often it's shared/read.

### Builders as immutability's practical companion
Because immutable objects can't be assembled incrementally (no setters to call step by step), classes with many fields are commonly paired with a separate mutable builder that accumulates values and produces one finished immutable object at the end:
```java
Account acc = new Account.Builder()
    .accountNumber("ACC123")
    .type(AccountType.IGSTK)
    .build();   // builder is mutable during assembly; the Account it produces is not
```

The builder itself is deliberately mutable (its whole job is being filled in step by step) — immutability is a property of the finished Account object, not of the process used to construct it.

## `record` classes — a concise built-in immutable type
- Java 14 introduced `record` classes, which are a concise way to define immutable data carriers. A record automatically provides:
  - Final fields for all components.
  - A canonical constructor that initializes all fields.
  - Implementations of `equals()`, `hashCode()`, and `toString()`
- Records are implicitly final and cannot be extended.
```java
record Account(String accountNumber, AccountType type, List<String> transactionIds) {
    Account {           // compact constructor — no parameter list repeated, no explicit field assignment needed
        transactionIds = List.copyOf(transactionIds);   // defensive copy in the canonical constructor
    }
}
```
- The canonical constructor can be customized to include defensive copying or validation logic, while still benefiting from the automatically generated methods and immutability guarantees of the record.
