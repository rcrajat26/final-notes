# Serialization
## What it is, and where it fits
- **Serialization** converts an object graph (the object plus everything reachable from it) into a byte stream. 
- **Deserialization** rebuilds an equivalent graph from those bytes. 
- Typical uses are writing objects to disk, sending them over a network, and storing them in caches or HTTP sessions.
- Java's built-in mechanism is binary and Java-specific. 
- Today it is mostly legacy: new systems use JSON, Protobuf or Avro. But we still need to know it because old codebases and some caches and session stores use it, and because its security failures are well known.

## The basics 
```java
import java.io.*;

public class Account implements Serializable {   // marker interface: no methods
    private static final long serialVersionUID = 1L;

    private final String accountId;
    private AccountType type;                    // enum: IGCFD / IGSTK
    private String status;
    // ...
}

// write
try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("client.bin"))) {
    out.writeObject(client);
}

// read
try (ObjectInputStream in = new ObjectInputStream(new FileInputStream("client.bin"))) {
    Client restored = (Client) in.readObject();   // cast needed; throws ClassNotFoundException
}
```

- `Serializable` is a marker interface. It has no methods and only tells the runtime that serialization is allowed. 
- If any object reachable from the root is not `Serializable`, writeObject throws `NotSerializableException` at runtime. The compiler cannot check this, because field types are often interfaces and the real object is only known at runtime. 
- readObject returns Object, so you cast. The class must be on the classpath, otherwise `ClassNotFoundException`.

## The whole graph is saved, with shared references and cycles preserved
- `Client` holds a `List<Account>`, and each `Account` points back to its `Client`. Serializing the `Client` serializes all of it.

```java
Client c = new Client("Asha", "asha@x.com", "555-0100");
Account a1 = new Account(c, AccountType.IGCFD);
Account a2 = new Account(c, AccountType.IGSTK);
c.addAccount(a1); c.addAccount(a2);
```

The stream writer keeps a handle table of every object it has already written. The second time it meets the same object, it writes a small back-reference instead of the object again. This has three consequences:
- Cycles (`Client` → `Account` → `Client`) do not cause infinite recursion. 
- Identity is preserved. After deserialization, `a1.getClient() == a2.getClient()` is still `true`, because both point to one restored Client. 
- Separate `writeObject` calls on the same stream share the table. Writing the same object twice produces a back-reference, and changes you made between the two writes are not re-sent. Use `out.reset()` to clear the table, or `writeUnshared`.

### `transient` and `static`
**Field Serialization Behavior**

| Field kind                 | Serialized? | After deserialization                         |
|----------------------------|-------------|-----------------------------------------------|
| Normal instance field      | Yes         | Restored                                      |
| `transient` instance field | No          | Default value (`null`, `0`, `false`)          |
| `static` field             | No          | Whatever the class currently holds in the JVM |

- Use `transient` for secrets (`passwordHash`, tokens), for derived or cached values (`cachedDisplayName`), and for non-serializable resources (`Connection`, `Thread`, `Logger`). 
- Trap: a transient field comes back as null or 0, not as its initializer value. Field initializers do not run during deserialization.

### Constructor behavior during serialization 
- This is the part most people get wrong. Deserialization creates the object without calling the constructor of the `Serializable` class. 
- The rule is by class hierarchy. Walk up from your class to the first non-serializable superclass:
  - The no-arg constructor of that first non-serializable superclass runs. It must be accessible, otherwise you get `InvalidClassException`. 
  - No constructor of any serializable class in the chain runs. Field initializers and instance init blocks of those classes don't run either. Their fields come from the stream. 
  - The fields of the non-serializable superclass are not in the stream. They hold whatever its no-arg constructor set.

```java
class BaseEntity {                       // NOT Serializable
    protected long createdAt = System.currentTimeMillis();
    BaseEntity() { System.out.println("BaseEntity()"); }   // runs on deserialization
}

class Account extends BaseEntity implements Serializable {
    private String status = "ACTIVE";    // initializer does NOT run on deserialization
    Account() { System.out.println("Account()"); }          // does NOT run
}
```

- On deserialization, `BaseEntity()` prints, `Account()` doesn't, and status comes from the stream. `createdAt` is re-initialized to the current time, not restored. 
- The consequence is that a constructor's validation does not protect a deserialized object. 
- The stream is the input, so a crafted stream can produce an object that violates your invariants. 
- This is the root of both the security risk and the immutability problems below.

### `serialVersionUID`: the compatibility check
- Each stream records the class's `serialVersionUID`. On read, the JVM compares it to the local class's value. A mismatch throws `InvalidClassException`. 
- If you don't declare it, the JVM computes it from a hash of the class's structure (name, modifiers, interfaces, fields, methods and so on). The result is fragile. Adding a method can change it, and different compilers can produce different values. Old data then fails to load after an innocent refactor.
- If you declare it (`private static final long serialVersionUID = 1L;`), you control compatibility. Bump it only when you deliberately want old data rejected.

What stays compatible with a fixed serialVersionUID, for the default mechanism:
## Serialization Compatibility

| Change | Result |
|---|---|
| Adding a field | Compatible. Old streams give the new field its default value. |
| Removing a field | Technically loads, but the value is dropped and old readers get defaults, so treat it as risky. |
| Changing a field's type | Incompatible (`InvalidClassException`) |
| Renaming or moving the class | Incompatible (the class name is in the stream) |
| Changing the class hierarchy | Incompatible |
| Switching between `Serializable` and `Externalizable` | Incompatible |


### Customizing the mechanism
Reference class:
```java
public class Account implements Serializable {
    private static final long serialVersionUID = 1L;

    private final String accountId;
    private AccountType type;
    private transient String passwordHash;        // handled manually below

    // ... constructors, getters ...

    // (a) hooks: private, in THIS class, handling THIS class's fields only
    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();
        out.writeObject(encrypt(passwordHash));
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        this.passwordHash = decrypt((String) in.readObject());
    }

    // (b) replacement hooks: also in this class
    private Object writeReplace() { return new Proxy(this); }
    private Object readResolve()  { return INSTANCE; }   // singleton case, not alongside the proxy

    // the Proxy is a private static nested class of the same class
    private static class Proxy implements Serializable { ... }
}
```
a) `writeObject` / `readObject` hooks
These are private methods with exact signatures. They are found by reflection, not through an interface.
```java
private void writeObject(ObjectOutputStream out) throws IOException {
    out.defaultWriteObject();                 // writes all non-transient, non-static fields
    out.writeObject(encrypt(passwordHash));   // extra custom data
}

private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
    in.defaultReadObject();                   // restores the normal fields
    this.passwordHash = decrypt((String) in.readObject());   // must read in the SAME order written
    validate();                               // re-check invariants (see section 9)
}
```

b) `writeReplace` / `readResolve`
`writeReplace()` lets an object substitute a different object to be written.
`readResolve()` is called on the freshly deserialized object, and its return value replaces it. It is the standard fix for singletons.
```java
public final class MarketConfig implements Serializable {
    private static final MarketConfig INSTANCE = new MarketConfig();
    private MarketConfig() {}
    public static MarketConfig getInstance() { return INSTANCE; }

    private Object readResolve() { return INSTANCE; }   // without this, deserialization makes a 2nd instance
}
```
Without `readResolve`, deserializing a `Serializable` singleton creates a second instance and breaks the singleton guarantee. The cause is that deserialization bypasses the private constructor.

c) `Externalizable` interface
```java
public class Quote implements Externalizable {
    public Quote() {}                         // REQUIRED: public no-arg constructor
    public void writeExternal(ObjectOutput out) throws IOException { ... }
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException { ... }
}
```

- You write and read every field by hand. Nothing is automatic and transient has no meaning. 
- On read, the public no-arg constructor runs first, then readExternal fills in the fields. This is the opposite of Serializable. 
- It can give a more compact format, but it costs more maintenance and is easier to get wrong.


## Security, and the immutability problem
**Why deserialization is dangerous**
- readObject() instantiates classes named in the stream and runs their readObject hooks. 
- If you deserialize attacker-controlled bytes, the attacker chooses which classes on your classpath get instantiated. 
- Chaining existing classes ("gadgets") can lead to remote code execution. This has been behind many real-world CVEs.

Defenses:
- Never deserialize untrusted data with native serialization. Use JSON, Protobuf or a similar data-only format.
- Serialization filters (ObjectInputFilter, Java 9, JEP 290): an allow-list or deny-list of classes, plus limits on depth, array size and number of references. Java 17 (JEP 415) added a JVM-wide filter factory. Use -Djdk.serialFilter=... or set a filter on the stream.
- Keep the classpath small, because every class on it is a potential gadget.

**Deserialization bypasses constructor validation**
- A class whose constructor checks balance >= 0 or makes a defensive copy gets none of that protection on the way in. 
- A forged stream can supply any field values. 
- It can also supply internal references that the attacker still holds, so a "private" Date or List becomes shared mutable state.

The fixes are:
1. Re-validate in readObject, and defensively copy mutable fields there too.
2. Serialization Proxy pattern, the recommended approach for sensitive classes: the real class serializes a small private static nested Proxy object instead (via writeReplace). The proxy's readResolve rebuilds the real object through the public constructor, so validation always runs. A readObject that throws prevents anyone from reading the real class's stream form directly.

```java
private Object writeReplace() { return new Proxy(this); }
private void readObject(ObjectInputStream in) throws InvalidObjectException {
    throw new InvalidObjectException("Proxy required");
}

private static class Proxy implements Serializable {
    private final String accountId; private final AccountType type;
    Proxy(Account a) { this.accountId = a.accountId; this.type = a.type; }
    private Object readResolve() { return new Account(accountId, type); }   // public constructor => validation runs
}
```


