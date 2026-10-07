# Null Discipline
## What null is
- `null` is the value a reference variable holds when it points to no object. 
- It is not an object, it has no type of its own, and it can be assigned to any reference type. 
- Primitives (int, boolean, ...) can never be null.

```java
Account a = null;       // legal: the variable exists, the object doesn't
int n = null;           // compile error
a.getStatus();          // NullPointerException at runtime: no object to call it on
```

- The compiler does not track nullability in the type system, so `Account` means "an `Account` or `null`". 
- Null discipline is therefore a set of conventions and habits that stand in for what the language doesn't enforce. 
- Fields of reference type default to `null`, and so do array elements: `new Account[5]` is five `null` slots. 
- Locals have no default and must be definitely assigned before use

### Every way we get an NPE
| Operation on a null reference                                  | Result |
|----------------------------------------------------------------|--------|
| Instance method call: `a.getStatus()`                          | `NPE`  |
| Instance field read/write: `a.status`                          | `NPE`  |
| Array access or length: `arr[0]`, `arr.length`                 | `NPE`  |
| Unboxing a null wrapper: `int x = nullInteger;`                | `NPE`  |
| `throw null;`                                                  | `NPE`  |
| `synchronized (nullRef) { }`                                   | `NPE`  |
| Enhanced `for` over a null array or iterable                   | `NPE`  |
| `switch` on a null `String`/enum/wrapper (without `case null`) | `NPE`  |

### Things that do not throw:
- null instanceof Account is false. 
- (Account) null is a legal cast.
- "ACTIVE" + null gives "ACTIVEnull".
- nullRef == null is a plain comparison.
- Calling a static method through a null reference works, because the compiler resolves it from the declared type and never touches the object: ((Account) null).staticHelper() runs fine. It is legal but misleading.


## Avoiding NPE 
### Fail fast: validate at the boundary
The most valuable habit is to reject null where it enters your object, not where it eventually blows up.
```java
public final class Account {
    private final String accountId;
    private final AccountType type;

    public Account(String accountId, AccountType type) {
        this.accountId = Objects.requireNonNull(accountId, "accountId must not be null");
        this.type      = Objects.requireNonNull(type, "type must not be null");
    }
}
```

## Null Validation in Constructors
| Without the check                             | With the check                                   |
|-----------------------------------------------|--------------------------------------------------|
| Constructor succeeds, a null field is stored. | Constructor throws immediately.                  |
| NPE appears later, maybe in another thread    | Stack trace points at the call that passed null. |
| Object lives in an invalid state meanwhile.   | Invalid object never exists.                     |

Where to check: public constructors and public methods, which are trust boundaries. Inside a class, private helpers can trust that the checks already happened. Re-checking at every layer is noise.

### Designing APIs: returning "nothing"
| Situation | Preferred approach |
|---|---|
| Method returns a collection or array | Return empty, never null. Callers can loop without a guard. |
| Method returns a string that might be absent | Decide: `""` if empty is a valid value, otherwise document `null`, or throw. |
| Absence means the caller made a mistake or the data is corrupt | Throw (`IllegalArgumentException`, or a domain exception such as `AccountNotFoundException`). |
| Absence is a normal, expected outcome of a lookup | Return an `Optional<T>`. It exists specifically so the return type says "might be absent". |
| A "do nothing" behavior is needed | A Null Object: a real object whose methods are harmless no-ops (e.g. `NoOpAuditLogger`), so callers need no `if`. |

### Avoid null parameters
- Avoid designs where callers pass null to mean "use the default" or "skip this". Prefer method overloading for parameters that might be absent, rather than accepting null.
- Document parameters that can be null using `@Nullable` or similar annotations.
```java
// Bad: null means "no limit"
void open(Client c, BigDecimal limit)

// Better: intent is explicit at the call site
void open(Client c)
void open(Client c, BigDecimal limit)    // limit must be non-null
```

### Documenting nullness: annotations
- The JDK has no standard `@Nullable`/`@NonNull`. Common choices are the JetBrains, Spring (`org.springframework.lang`), Jakarta, SpotBugs and Checker Framework annotations. Most teams pick one and apply it consistently.
```java
public @Nullable Account findPrimary(@NonNull Client client) { ... }
```

## Comparing safely
- Put the known non-null value first, or use Objects.equals:
```java
if (status.equals("ACTIVE"))   // NPE if status is null
if ("ACTIVE".equals(status))   // safe: literal can't be null
if (Objects.equals(a, b))      // safe on either side; two nulls are equal
```

Related facts:
- `equals(null)` must return false, never throw (part of the equals contract).
- `compareTo(null)` is allowed to throw NPE. The Comparable contract says so.
- `Objects.hashCode(null)` returns 0. `Objects.hash(a, b)` tolerates null components.
- `Objects.toString(obj, "N/A")` gives a default for null.
- Enums: `status == AccountStatus.ACTIVE` is null-safe, while `status.equals(...)` is not. `==` on enums is both safe and idiomatic.


## `null` in `switch` (Java 21)
- In Java 21, switch expressions and statements can handle null values safely. You can use a null case label to explicitly handle null values.
```java
String status = null;
switch (status) {
    case "ACTIVE" -> System.out.println("Active");
    case "INACTIVE" -> System.out.println("Inactive");
    case null -> System.out.println("Status is missing");
    default -> System.out.println("Unknown status");
}
```
Without `case null`, a null selector still throws NPE, so existing code is unchanged.

## Subtle traps with `null`
### Conditional-operator unboxing. 
- When one branch is a primitive and the other a wrapper, the result type becomes the primitive, so a null wrapper gets unboxed:
```java
Integer limit = null;
int x = flag ? 100 : limit;      // flag == false: NPE (limit is unboxed)
Integer y = flag ? 100 : null;   // fine: both operands are reference-compatible
```

### Overload resolution with a null literal.
- When a method is overloaded, passing null can be ambiguous if multiple overloads accept reference types. The compiler will choose the most specific applicable overload, but if there are multiple equally specific candidates, it results in a compile-time error.
```java
void log(Object o) { }
void log(String s) { }
log(null);                       // picks log(String): most specific wins

void audit(String s) { }
void audit(Integer i) { }
audit(null);                     // COMPILE ERROR: ambiguous
```

## Chains and what to do about them
```java
String t = client.getPrimaryAccount().getType().name();   // three possible NPE points
```

- Java has no safe-navigation operator (`?.`)
- Design so the chain can't be null: non-null constructor-validated fields, empty collections, never-null getters.
- Give the owning class a method that does the work (client.primaryAccountTypeName()), so callers don't traverse the structure themselves. 
- Guard explicitly with if, once, at the right level. 
- Optional-returning getters (flagged only).

Do not wrap the chain in `try { } catch (NullPointerException e) { }`. That hides real bugs, makes the failure path unobservable, and mixes control flow with error handling. Treat an NPE as a programming defect to fix, not a condition to handle.

### Trap summary
- Primitives can't be null, but unboxing a null wrapper is an NPE, and ternaries can unbox silently.
- Fields and array elements default to null. Locals have no default and need definite assignment.
- static methods called via a null reference run fine, because the declared type is used.
- null instanceof X is false. Casting null is legal.
- x.equals(literal) is NPE-prone. Put the non-null value first, or use Objects.equals.
- compareTo(null) may throw. equals(null) must return false.
- switch on null throws NPE unless case null is present (Java 21).
- log(null) can silently pick the most specific overload, or fail to compile if two are equally specific.
- Map.get returning null is ambiguous between "absent" and "mapped to null".
- List.of(...).contains(null) throws. The immutable factories reject nulls.
- Never return null for collections or arrays.
- Catching NullPointerException hides defects. Validate early instead.
- Nullability annotations are metadata only. The JVM never enforces them.
- Helpful NPE messages are Java 14+ (on by default from 15). Local variable names need -g.
- Optional is for return types only, never for fields, parameters or elements. Never assign null to an Optional variable.

---

# Miscellaneous
## Null-Safety Quick Reference

| Question                                           | Answer                                         |
|----------------------------------------------------|------------------------------------------------|
| Constructor/public method receives a required arg? | `Objects.requireNonNull(arg, "msg")`           |
| Returning a collection with no results?            | Empty collection, not `null`                   |
| Absence is exceptional/error?                      | Throw a meaningful exception                   |
| Absence is a normal lookup outcome?                | `Optional` return (flagged)                    |
| Need to compare with a literal?                    | `"LITERAL".equals(x)`                          |
| Comparing enums?                                   | `==`                                           |
| Optional behavior, "do nothing" case?              | Null Object                                    |
| Want tooling to catch it?                          | Nullability annotations plus a static analyzer |
| Tempted to catch NPE?                              | Fix the cause instead                          |


## Returning `Optional` 
- `Optional<T>` was designed for one job: telling the caller "this method might legitimately have no result.", Instead of returning null and hoping the caller remembers to check. `Optional<User> findByEmail(String email);` The caller can't directly treat this as a user and has to unwrap it.
- Optional is not meant for member fields, and avoid below:
```java
class Order {
    private Optional<Coupon> coupon;   // avoid
    private Coupon coupon;             // fine, @Nullable or documented
}
```
- Optional is not meant for method parameters, and avoid below:
```java
class OrderService {
    void applyCoupon(Optional<Coupon> coupon) {   // avoid
        ...
    }
    void applyCoupon(Coupon coupon) {             // fine, @Nullable or documented
        ...
    }
}
```
- Optional is not meant for collection elements, and avoid below:
```java
class Order {
    private List<Optional<Coupon>> coupons;   // avoid
    private List<Coupon> coupons;             // fine, @Nullable or documented
}
```
- Optional itself shouldn't be null hence never return null in a scenario where we are returning an optional variable: 
```java
Optional<User> findByEmail(String email) {
    return null;                  // never do this
    return Optional.empty();      // do this
}
```
