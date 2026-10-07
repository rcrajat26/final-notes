# Design Idioms
## Static factory methods
- A `static` factory method is a `static` method that returns an instance of its class (or a subtype), offered instead of, or alongside, a public constructor.
```java
public final class Account {
    private final String accountId;
    private final AccountType type;
    private final BigDecimal balance;

    private Account(String accountId, AccountType type, BigDecimal balance) {
        this.accountId = Objects.requireNonNull(accountId);
        this.type      = Objects.requireNonNull(type);
        this.balance   = Objects.requireNonNull(balance);
    }

    public static Account openCfd(String id)   { return new Account(id, AccountType.IGCFD, BigDecimal.ZERO); }
    public static Account openStock(String id) { return new Account(id, AccountType.IGSTK, BigDecimal.ZERO); }
}
```

### Advantages of Static Factory Methods over Constructors
| Advantage | Explanation |
|---|---|
| They have names | `Account.openCfd(id)` says what it does. Constructors are all named after the class, and two with the same parameter types can't coexist. |
| No new object per call | A factory can return a cached or shared instance. `Boolean.valueOf(true)` always returns `Boolean.TRUE`, and `Integer.valueOf` uses its small-value cache. A constructor must always allocate. |
| Can return a subtype | The declared return type can be an interface or superclass while the real object is a hidden class. `List.of(...)` returns an internal immutable implementation you can't name. |
| Return type can vary by input | `EnumSet.noneOf(...)` returns one of two internal classes depending on the enum's size, and the caller never knows. |
| The implementation class can change later | Callers depend only on the declared type, so you can swap the concrete class without breaking them. |
| Can fail or return without constructing | Validate, return a shared constant, or return an existing instance. |

### Common Static Factory Method Naming Conventions
| Name                            | Typical meaning                          | Example                                |
|---------------------------------|------------------------------------------|----------------------------------------|
| `of(...)`                       | Aggregate or create from the given parts | `List.of`, `LocalDate.of(2026, 10, 6)` |
| `from(...)`                     | Convert from another type                | `Date.from(instant)`                   |
| `valueOf(...)`                  | Convert or look up, often cached         | `Integer.valueOf("5")`                 |
| `parse(...)`                    | Build from text                          | `LocalDate.parse("2026-10-06")`        |
| `getInstance()`                 | Return an instance, possibly shared      | `Calendar.getInstance()`               |
| `newInstance()` / `create...()` | Always a fresh instance                  | `Array.newInstance(...)`               |

## Builder
A class with many parameters, several optional, is awkward with both obvious alternatives:

| Approach | Problem |
|---|---|
| Telescoping constructors (`Client(name)`, `Client(name, email)`, `Client(name, email, phone)`, ...) | Combinatorial growth. Call sites become `new Client("Asha", null, "555", null, true)`, and swapping two same-typed arguments compiles silently. |
| No-arg constructor plus setters | The object exists half-built between calls, so it can be seen in an invalid state. It can't be immutable, because fields can't be `final`, and there is no single point at which to validate everything together. |


- The Builder pattern is a creational design pattern that allows for the step-by-step construction of complex objects. 
- It separates the construction of an object from its representation, allowing the same construction process to create different representations.
- This is particularly useful when an object has many optional parameters, or when the construction process involves multiple steps that can be chained together.
- A mutable `builder` collects the values step by step. A final `build()` call validates them together and produces an immutable object.

### Example: Builder Pattern for Account
```java
public final class Client {
    private final String name;          // required
    private final String email;         // required
    private final String phone;         // optional
    private final List<Account> accounts;

    private Client(Builder b) {
        this.name     = b.name;
        this.email    = b.email;
        this.phone    = b.phone;
        this.accounts = List.copyOf(b.accounts);          // defensive, immutable copy
    }

    public static Builder builder(String name, String email) {   // required fields up front
        return new Builder(name, email);
    }

    public static final class Builder {                    // static nested: no outer instance needed
        private final String name;
        private final String email;
        private String phone;
        private final List<Account> accounts = new ArrayList<>();

        private Builder(String name, String email) {
            this.name  = Objects.requireNonNull(name);
            this.email = Objects.requireNonNull(email);
        }
        public Builder phone(String phone)       { this.phone = phone; return this; }
        public Builder addAccount(Account a)     { accounts.add(Objects.requireNonNull(a)); return this; }

        public Client build() {
            if (!email.contains("@")) throw new IllegalStateException("invalid email");
            return new Client(this);
        }
    }
}

Client c = Client.builder("Asha", "asha@x.com")
                 .phone("555-0100")
                 .addAccount(Account.openCfd("A-1"))
                 .build();
```

**Points that matter:**
- The builder is a `static` nested class. It needs no reference to an outer instance, and it can reach the outer class's private constructor.
- Each setter-style method returns `this`, which is what makes the call chain ("fluent interface") possible.
- Required values go into the builder's constructor or factory. Optional ones are methods. A missing required value is then a compile error rather than a runtime surprise.
- `build()` is the one place for cross-field validation. Validate after copying the builder's values into the new object, not before, since a builder is mutable and can be altered between validation and copy (the same time-of-check/time-of-use issue as with any mutable input).
- Copy mutable collections when building (`List.copyOf`). Sharing the builder's list would let later builder calls change an "immutable" object.
- A builder is single-purpose and not thread-safe. Create one, populate it, build, discard. It can be reused to produce several objects, but only if `build()` copies everything.

### Choosing a Construction Approach
| Situation                                                      | Choice                              |
|----------------------------------------------------------------|-------------------------------------|
| 1 to 3 parameters, all required                                | Plain constructor or static factory |
| Several optional parameters, or same-typed adjacent parameters | Builder                             |
| Fixed small set of meaningful variants                         | Several named static factories      |
| Simple immutable data carrier                                  | `record`                            |

## Singleton
- A singleton is a design pattern that restricts the instantiation of a class to one "single" instance. This is useful when exactly one object is needed to coordinate actions across the system.
- The singleton pattern ensures that a class has only one instance and provides a global point of access to that instance.
- In Java, a singleton can be implemented in several ways, including using a private constructor, a static factory method, or an enum.

### Form 1: eager static final field
```java
public final class MarketConfig {
    private static final MarketConfig INSTANCE = new MarketConfig();
    private MarketConfig() {}
    public static MarketConfig getInstance() { return INSTANCE; }
}
```

- The instance is created when the class is initialized, which happens on first active use of the class (first `getInstance()` call), not at JVM start. 
- The JVM guarantees class initialization runs once and is visible to all threads, so this is safe without any extra code.

### Form 2: lazy holder (initialization-on-demand)
```java
public final class MarketConfig {
    private MarketConfig() {}
    private static class Holder {
        static final MarketConfig INSTANCE = new MarketConfig();
    }
    public static MarketConfig getInstance() { return Holder.INSTANCE; }
}
```

- `Holder` is not loaded or initialized until `getInstance()` touches it. 
- Creation is lazy, and thread safety comes from the JVM's class initialization guarantee, with no explicit locking. 
- It is the preferred lazy form when the singleton needs expensive construction or when the outer class has other static uses that shouldn't trigger it.

### Form 3: synchronized accessor (correct but slow)
```java
private static MarketConfig instance;
public static synchronized MarketConfig getInstance() {
    if (instance == null) instance = new MarketConfig();
    return instance;
}
```

- Correct, but every call acquires the lock, even long after initialization.

### Form 5: double-checked locking (DCL): broken unless `volatile`
Attempt to avoid locking on every call:
```java
private static MarketConfig instance;          // NOT volatile: BROKEN

public static MarketConfig getInstance() {
    if (instance == null) {                    // first check, no lock
        synchronized (MarketConfig.class) {
            if (instance == null) {            // second check, with lock
                instance = new MarketConfig();
            }
        }
    }
    return instance;
}
```

- **Why it breaks** `instance = new MarketConfig()` is not one atomic step. Conceptually it is:
```
1. allocate memory for the object
2. run the constructor (initialize fields)
3. store the address into `instance`
```

- The compiler and CPU are allowed to reorder 2 and 3 when no ordering is enforced. 
- Thread A could publish the address (step 3) before the constructor has finished. 
- Thread B then passes the unlocked first check, sees a non-null instance, and uses an object whose fields are still default values. 
- The unlocked read doesn't synchronize with anything, so nothing guarantees B sees A's writes in order.

**The fix**: declare the field volatile. This forbids the reorder and makes the write visible to readers:
```java
private static volatile MarketConfig instance;
```

### Singleton Attack Vectors and Defenses

| Attack | How it happens | Defense |
|---|---|---|
| Reflection | `setAccessible(true)` on the private constructor, then `newInstance()` makes a second object | Throw in the constructor if an instance already exists. The enum form is immune. |
| Serialization | Deserialization bypasses the private constructor and builds a new object | Add `private Object readResolve() { return INSTANCE; }`. The enum form is immune. |
| Cloning | If the class implements `Cloneable`, `clone()` copies it | Don't implement `Cloneable`, or throw from `clone()` |
| Multiple class loaders | The same class loaded by two loaders has two sets of statics, so two "singletons" | Inherent to the design: a singleton is one per class loader, not per JVM or per application |

Costs of singletons as a design
- They are global mutable state behind a respectable name.
- Callers have a hidden dependency on them, so it doesn't appear in any constructor signature.
- Testing is harder: you can't substitute a fake without extra machinery, and state leaks between tests.
- In a DI container (Spring, for instance), a "singleton-scoped bean" means one instance per container, created and passed in by the container. It is an ordinary class with a public constructor, with none of the above problems. In that world, hand-written singletons are rarely needed. Use the idiom for genuinely process-wide, stateless or read-only things, and let the container manage the rest.

