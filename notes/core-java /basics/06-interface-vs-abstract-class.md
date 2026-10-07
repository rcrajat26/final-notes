# Interfaces vs Abstract Classes

| Feature                   | Abstract class                              | Interface (Java 8+)                                                         |
|---------------------------|---------------------------------------------|-----------------------------------------------------------------------------|
| Keyword                   | `abstract class`                            | `interface`                                                                 |
| Multiple inheritance      | ❌ Only one `extends`                       | ✅ A class can `implements` many                                            |
| Instance fields (state)   | ✅ Yes                                      | ❌ No — only `public static final` constants                                |
| Constructors              | ✅ Yes (called via subclass's `super(...)`) | ❌ No constructors at all                                                   |
| Method bodies             | ✅ Always could                             | ✅ Since Java 8, via `default`/`static`/`private`                           |
| Can restrict implementers | ✅ Via `sealed` (Java 17)                   | ✅ Via `sealed` (Java 17)                                                   |
| Diamond conflict possible | N/A (single inheritance)                    | ✅ Possible with conflicting defaults — compiler forces explicit resolution |

- Every field in an interface is implicitly `public static final`, irrespective of whether we write it or not:
```java
public interface Auditable {
    int MAX_RETRY_ATTEMPTS = 3;   // implicitly: public static final int MAX_RETRY_ATTEMPTS = 3;
    void generateAuditReport();
}
```

- This reinforces: interfaces define behavior contracts, never state.

### When to pick which — practical decision framework
Ask, in order:
1. Does this represent an "is-a" relationship with shared state and partial shared behavior?
   → Abstract class. Example: Account — every subtype genuinely is an account, shares status/type fields, and shares real logic like activateAccount().
2. Does this represent "can do X," independent of what the class otherwise is?
   → Interface. Example: Auditable — a Client and an Account share no inheritance relationship, but both can independently promise "I can be audited."
3. Does a class need to satisfy multiple unrelated contracts at once?
   → Must be interfaces — Java physically does not allow extending more than one class. PremiumClient extending Client while also promising Auditable and Serializable is only possible because the latter two are interfaces.
4. Is this just a reusable implementation detail with no real "is-a" meaning?
   → Neither — use composition. This is exactly the SurchargeRule/CountryMatcher situation: SurchargeRule isn't "a kind of" matcher, so it holds a CountryMatcher field rather than inheriting from anything.


| Version            | What changed                                                                                                                                         |
|--------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| Java 7 and earlier | Interfaces = only `public static final` fields + `public abstract` methods. Zero implementation allowed. This is the "classic" interface we covered. |
| Java 8             | Added `default` and `static` methods to interfaces — interfaces can now have real method bodies.                                                     |
| Java 9             | Added `private` and `private static` methods to interfaces — for internal helper logic shared between default methods.                               |
| Java 17            | Sealed classes/interfaces finalized — lets you restrict exactly which classes may implement/extend a type.                                           |

**Default methods**: Why this was added: before Java 8, adding a new method to an interface caused every existing implementing class would suddenly fail to compile, because it wouldn't have that method. default methods let the JDK evolve interfaces (famously, adding .stream() to Collection) without breaking every existing implementation — classes that don't override the default method just inherit the given behavior automatically.

**Private methods**: Lets two default methods share common logic without exposing that helper as part of the public contract. Before Java 9, any shared logic between default methods had to either be duplicated or exposed as a public default method — leaking implementation detail into the interface's contract.


**Diamond problem with default methods**:
The resolution rules, in priority order (know this cold for interviews):
- A class's own method always wins over any interface default.
- The most specific interface wins — if B extends A and both have greet(), B's version is used automatically (no ambiguity, no error).
- If two unrelated interfaces both provide a default for the same method, it's a compile error — you must override and manually disambiguate with InterfaceName.super.method().

> "default methods don't bring back true multiple inheritance of state — there's still no shared mutable state — but they do reintroduce a narrow version of the multiple-inheritance-of-behavior problem, which Java resolves with mandatory explicit disambiguation rather than silent rules."

**`sealed` interface**
```java
public sealed interface CountryMatcher permits AllCountriesMatcher, CountryListMatcher, ExcludedCountriesMatcher {
    boolean accept(String condition);
}
```

- `sealed` lets you restrict exactly which classes are allowed to implement an interface (or extend a class) — listed explicitly via permits. 
- Before this, any interface could be implemented by any class, anywhere, with no way to close that off.

### Resolution rules in Java 
**Rule 1: Classes always win over interfaces**
If a concrete implementation exists anywhere in the superclass chain, it beats any default method from an interface — no matter how "closer" the interface seems. ex: `B` and `C` beat `A`

**Rule 2: The most specific interface wins**
If there's no class implementation, and multiple interfaces provide a default, the compiler picks the one from the most derived interface — i.e., if one interface extends (and overrides) the other, the subinterface's version wins automatically, no error. Ex: there is interface A. Interface B extends A. Class D implements both A and B. B's method wins as it is the immediate parent.

**Rule 3: True conflict → compiler forces explicit resolution**
When two unrelated interfaces each independently override the same default, there is no "most specific" one to pick — so Java refuses to guess and throws a compile error:

```java
interface B {
    default void greet() { System.out.println("Hello from B"); }
}
interface C {
    default void greet() { System.out.println("Hello from C"); }
}

class D implements B, C {
    // COMPILE ERROR:
    // class D inherits unrelated defaults for greet() from types B and C
}

//Correct version
class D implements B, C {
    @Override
    public void greet() {
        B.super.greet();  // explicit choice
        C.super.greet();
        System.out.println("Hello from D");
    }
} 
```
