# `equals`, `hashCode`, `==`, and `Object` Methods
## `==` behavior
- == — the one thing it always means
- == on object references always compares reference identity (same heap address) — never content, regardless of the class. This never changes, there's no overriding this. 
- For primitives, == compares value directly.

```java
Client c1 = new Client("Arjun", "a@x.com");
Client c2 = new Client("Arjun", "a@x.com");
System.out.println(c1 == c2); // false — two distinct heap objects, same content
```

## `equals()` — default behavior, and why you override it
- `Object.equals(Object o)`'s default implementation is literally `return this == o;` — identity comparison. 
- So every class starts out with `==`-equivalent `equals()` until you override it. 
- You override `equals()` when "logically the same" should mean "same content," not "same object" — e.g. two Client objects with the same email should probably be considered equal even though they're different heap objects.

```java
class Client {
    private String name;
    private String email;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                    // fast path — same reference
        if (o == null || getClass() != o.getClass()) return false; // trap below
        Client other = (Client) o;
        return Objects.equals(name, other.name) && Objects.equals(email, other.email);
    }
}
```

- `getClass() != o.getClass()` vs `o instanceof Client` — a real, meaningful choice, not stylistic:
  - `instanceof` allows a subclass instance to be `equals()` to a superclass instance (if fields match) — but this risks breaking symmetry (`a.equals(b)` must equal `b.equals(a)`).
  - `getClass()` comparison is stricter — an object is only equal to another object of the exact same runtime class, sidestepping the symmetry-breaking risk entirely.
- `Objects.equals(a, b)` (not deep in Collections territory, just a java.util.Objects utility method) — null-safe wrapper equivalent to a == null ? b == null : a.equals(b). Avoids writing manual null-checks for every field.


## The equals/hashCode contract — why they're a pair, not independent
- `Object.hashCode()`'s default implementation is typically derived from the object's memory address/identity (not specified to be literally the address, but behaves that way by default — distinct from identity itself, but correlated with it).
- The contract:
  - If two objects are equal according to `equals()`, they **must** have the same `hashCode()`.
  - If two objects are not equal, they **may** have the same `hashCode()` (hash collisions are allowed).
- This is crucial for hash-based collections like `HashMap` and `HashSet`. If you override `equals()` but not `hashCode()`, two logically equal objects could end up in different hash buckets, breaking the collection's behavior.
- `hashCode()` must be consistent — calling it multiple times on the same (unmodified) object returns the same value within one execution.

**Why rule 1 is non-negotiable — mechanism, not convention**: 
- hash-based structures use `hashCode()` to pick a bucket, then `equals()` to confirm the exact match within that bucket. 
- If two objects are `equals()` but have different `hashCode()`s, a lookup can land in the wrong bucket entirely and never even reach the `equals()` check — the object silently "disappears" from the structure's perspective, even though it's logically present. 
- This is why overriding `equals()` without also overriding `hashCode()` is one of the most common real-world Java bugs — compiles fine, works in simple tests, quietly breaks the moment the object is used as a `HashMap` key or put in a `HashSet`.

```java
@Override
public int hashCode() {
    return Objects.hash(name, email); // combines both fields' hash codes into one — must use the SAME fields as equals()
}
```

> Critical rule: hashCode() must be computed from the same fields equals() uses for comparison (a subset is technically contract-legal — fewer fields in hash, more buckets collide but contract holds; the reverse, more fields in hash than in equals, breaks the contract). The cleanest and safest default: same exact field set in both.

## `toString()` behavior
- Default is `ClassName@hexHashCode` (e.g. `Client@7a81197d`) — the hex part is literally `Integer.toHexString(hashCode())`, so an unoverridden `toString()` leaks exactly the identity-ish default hash code. Override for anything you'll log/print/debug:

```java
@Override
public String toString() {
    return "Client{name='" + name + "', email='" + email + "'}";
}
```

## Miscellaneous
### `Object` methods you should know about
| Method           | One-liner                                                                                |
|------------------|------------------------------------------------------------------------------------------|
| `equals(Object)` | Logical equality check — default is identity (`==`), override for content equality       |
| `hashCode()`     | hash-code for the object — must stay consistent with `equals()`                          |
| `toString()`     | String representation — default is `ClassName@hex`, override for anything human-readable |
| `getClass()`     | Returns the object's actual runtime `Class`                                              |
| `clone()`        | Shallow field-copy without calling a constructor — outdated/broken pattern               |
| `wait()`         | Pauses the current thread until notified — thread coordination                           |
| `notify()`       | Wakes one waiting thread — thread coordination                                           |
| `notifyAll()`    | Wakes all waiting threads — thread coordination                                          |
| `finalize()`     | Pre-GC cleanup hook — deprecated since Java 9, unreliable, not worth using               |


### `Objects` utility class methods
| Method                           | Signature (informal)                            | What it does                                                                                                                          |
|----------------------------------|-------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| `equals(a, b)`                   | `boolean equals(Object a, Object b)`            | Null-safe equality: `true` if both null, `false` if exactly one is null, otherwise `a.equals(b)`                                      |
| `deepEquals(a, b)`               | `boolean deepEquals(Object a, Object b)`        | Like `equals`, but if both are arrays, recursively compares contents (like `Arrays.deepEquals`) instead of comparing array references |
| `hashCode(o)`                    | `int hashCode(Object o)`                        | Null-safe hash: `0` if `o` is null, else `o.hashCode()`                                                                               |
| `hash(values...)`                | `int hash(Object... values)`                    | Combines multiple values' hash codes via `Arrays.hashCode` — covered in detail just above                                             |
| `toString(o)`                    | `String toString(Object o)`                     | Null-safe `toString()`: returns `"null"` (the string) if `o` is null, else `o.toString()`                                             |
| `toString(o, default)`           | `String toString(Object o, String nullDefault)` | Same, but returns your own specified fallback string instead of `"null"` if `o` is null                                               |
| `isNull(o)`                      | `boolean isNull(Object o)`                      | `true` if `o == null` — exists mainly to use as a method reference (`Objects::isNull`) in a filter/predicate context                  |
| `nonNull(o)`                     | `boolean nonNull(Object o)`                     | Opposite of `isNull` — same method-reference motivation                                                                               |
| `requireNonNull(o)`              | `T requireNonNull(T o)`                         | Returns `o` if not null; throws `NullPointerException` if it is. The standard, idiomatic way to validate constructor/method arguments |
| `requireNonNull(o, message)`     | `T requireNonNull(T o, String message)`         | Same, but with a custom exception message — far more debuggable than a bare NPE                                                       |
| `requireNonNullElse(o, default)` | `T requireNonNullElse(T o, T default)`          | Returns `o` if non-null, else returns `default` — a null-coalescing helper (Java 9+)                                                  |
| `compare(a, b, comparator)`      | `int compare(T a, T b, Comparator<T> c)`        | Returns `0` immediately if `a == b` (identity short-circuit), else delegates to the given comparator                                  |

Example: `requireNonNull` use case
```java
class Client {
    private final String name;
    private final String email;

    Client(String name, String email) {
        this.name = Objects.requireNonNull(name, "name must not be null");
        this.email = Objects.requireNonNull(email, "email must not be null");
    }
}
```

