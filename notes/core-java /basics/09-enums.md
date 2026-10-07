## Enums

### What an enum actually is under the hood

```java
enum AccountStatus {
    ACTIVE, PENDING, SUSPENDED, CLOSED
}
```

- An enum is **not a special JVM construct** — it's compiler sugar over a class. 
- `AccountStatus` compiles to a `final class AccountStatus extends java.lang.Enum<AccountStatus>`, with each constant (`ACTIVE`, `PENDING`, ...) compiled into a `public static final AccountStatus` field. 
- Each constant is a genuine singleton object, constructed exactly once, at class-load time, in a static initializer the compiler writes for you.

- **`final` implicitly**: you cannot extend an enum (`extends` is already consumed by `java.lang.Enum`).
- **Constructors are implicitly `private`** — you can never call `new AccountStatus(...)` yourself; only the compiler-generated static block does, once per constant, at class load.

### Fields, constructors, methods on enums — domain example

```java
enum AccountType {
    IGCFD(5000, true),
    IGSTK(10000, false);

    private final int minBalance;
    private final boolean marginEligible;

    AccountType(int minBalance, boolean marginEligible) {
        this.minBalance = minBalance;
        this.marginEligible = marginEligible;
    }

    int getMinBalance() { return minBalance; }
    boolean isMarginEligible() { return marginEligible; }
}
```
- Each constant implicitly calls the matching constructor with the arguments listed right after its name — `IGCFD(5000, true)` runs the constructor once, for that one singleton object.
- Fields behave exactly like any class's fields — can be `final`, can have per-constant values, as above.

### Inherited methods every enum constant gets for free (from `java.lang.Enum`)

- **`name()`** — exact declared constant name as a `String` (`"IGCFD"`). Final, cannot be overridden.
- **`ordinal()`** — zero-based declaration-order position (`IGCFD` → 0, `IGSTK` → 1). **Trap**: ordinal shifts if you reorder/insert constants later — never persist `ordinal()` to a database or wire format; it's a position, not an identity.
- **`toString()`** — defaults to same as `name()`, but *can* be overridden (unlike `name()`), e.g. for a nicer display string.
- **`compareTo()`** — compares by ordinal (declaration order). Enums implement `Comparable` automatically.
- **`values()`** — static method (compiler-generated, not inherited from `Enum`), returns a `new` array of all constants in declaration order every time it's called.
- **`valueOf(String)`** — static, looks up a constant by exact `name()` string; throws `IllegalArgumentException` if no match.

### Switch over enums — case labels, no qualification

```java
switch (status) {
    case ACTIVE:    ...
    case SUSPENDED: ...
    default:        ...
}
```
Case labels are bare constant names (`ACTIVE`, not `AccountStatus.ACTIVE`) — the compiler already knows the switch's type is `AccountStatus`, qualifying would be redundant and is in fact illegal syntax here.

### Enums implementing interfaces — per-constant behavior override

```java
interface RiskPolicy {
    double marginMultiplier();
}

enum AccountType implements RiskPolicy {
    IGCFD {
        public double marginMultiplier() { return 0.05; }
    },
    IGSTK {
        public double marginMultiplier() { return 0.10; }
    };
}
```
- An enum can `implements` any number of interfaces — just can't `extends` anything beyond the implicit `Enum`.
- Each constant can supply its **own body**, overriding a method differently per constant.

### Trap summary (quick reference)

| Trap                            | Rule                                                                         |
|---------------------------------|------------------------------------------------------------------------------|
| `ordinal()` persistence         | Never store ordinal in DB/wire format — reorder-fragile                      |
| `values()` cost                 | Allocates a new array every call — cache it yourself if called in a hot loop |
| `valueOf()` miss                | Throws `IllegalArgumentException`, not NPE/ClassCastException                |

---

