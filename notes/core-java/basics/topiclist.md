# Core Java Basics: Topic List (gaps still to cover)

Scope: core Java for interview prep. Out of scope: multi-threading, collections, generics, modern Java.
Deliberately excluded by the author: types and type system, operators, control flow, casting/`instanceof`, overload resolution/binding, file I/O, `Math`, `System` utilities, `this()`/`super()` chaining, enum serialization, return-in-`finally`, `@Inherited`.
Covered in later modules: GC, regex, common design patterns.

## A. New notes to write

| # | Topic | Must cover | Home |
|---|---|---|---|
| 1 | Object copying | `Cloneable` and `Object.clone()` (shallow copy, no constructor call, `CloneNotSupportedException`), shallow vs deep copy, why `clone` is considered broken, copy constructor, copy factory, covariant `clone()` override, arrays as the one safe `clone()` | New note, or a section in `10-equals-hashcode-==-object.md` (replaces its one-line `clone()` row) |

## B. Additions to existing notes

| # | Note | Add |
|---|---|---|
| 2 | `04-classes-objects.md` | **Private constructors**: uses (singleton, utility class, static-factory-only, builder-only), how it blocks `new` and subclassing |
| 3 | `04-classes-objects.md` | **Copy constructor**: syntax, shallow vs deep, link to topic 1 |

## C. Open: raised in review, not yet confirmed in or out

These came out of the first review and were not covered by the exclusions above. Decide each as keep or drop.

| Note | Possible addition |
|---|---|
| `03-arrays` | `System.arraycopy` is excluded. Remaining: `clone()` on arrays, `NegativeArraySizeException`, default element values |
| `05-oops-four-pillars` | `super` keyword usage, abstract class rules (no instantiation, constructors allowed) |
| `06-interface-vs-abstract-class` | Marker interfaces, functional interface basics |
| `06a-diamond-problem` | Largely duplicates `06`. Merge or trim |
| `07-final-static-modifiers` | Blank `final`, `final` parameters, `final` vs `finally` vs `finalize`, static-init rules (very thin note) |
| `08-nested-inner-local-anonymous-class` | `Outer.this`, shadowing, `Outer$Inner` compiled names, anonymous class vs lambda, inner-class memory leak |
| `10-equals-hashcode-==-object` | Full equals contract (reflexive, symmetric, transitive, consistent, null), mutable-key-in-hash-structure trap |
| `12-exceptions` | Suppressed exceptions, try-with-resources details (multiple resources, close order, Java 9 effectively-final resources), common exceptions list |
| `14-java-pass-by-value` | Swap trap, String/array/wrapper examples |
| `16-wrapper-boxing` | Cross-type `equals` (`Long.equals(Integer)`), `parseInt` vs `valueOf` |
| `19-annotations` | `@Documented` (the "four meta-annotations" wording is logged in `errors.md`) |
| New small topics | `Comparable` vs `Comparator`, `Locale`/`ResourceBundle`, `Properties`, `UUID`, `printf`/`Formatter`, `Random`/`SecureRandom`, `Runtime` shutdown hooks, variable scope/shadowing/`var`, logging basics |
