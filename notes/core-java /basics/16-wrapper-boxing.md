# Wrapper Classes & Boxing Internals
## What wrapper classes are, and why they exist
- Every primitive type has a corresponding wrapper class — an ordinary object wrapping a single primitive value inside it:

| Primitive | Wrapper class |
|-----------|---------------|
| `int`     | `Integer`     |
| `double`  | `Double`      |
| `boolean` | `Boolean`     |
| `char`    | `Character`   |
| `long`    | `Long`        |
| `short`   | `Short`       |
| `byte`    | `Byte`        |
| `float`   | `Float`       |


- Primitives aren't objects — they have no methods, can't be null, and can't be used anywhere Java's type system specifically requires an Object (generic type parameters, most collection APIs). 
- Wrapper classes exist to bridge exactly that gap: an Integer is a real object, sits on the heap, can be null, has methods (`Integer.parseInt(...)`, `Integer.compareTo(...)`, etc.), and can be stored anywhere an Object is expected.

### Autoboxing and unboxing — what the compiler actually does for you
```java
Integer boxed = 5;        // autoboxing — compiler rewrites this to: Integer.valueOf(5)
int unboxed = boxed;       // auto-unboxing — compiler rewrites this to: boxed.intValue()
```

### `Integer.valueOf(...)` vs `new Integer(...)` — and the integer cache
```java
Integer a = Integer.valueOf(100);
Integer b = Integer.valueOf(100);
System.out.println(a == b);    // true

Integer c = Integer.valueOf(200);
Integer d = Integer.valueOf(200);
System.out.println(c == d);    // false
```

- `Integer.valueOf(int)` is the preferred way to get an Integer object.
- As `Integer.valueOf(int)` maintains an internal cache of pre-created `Integer` objects for the range -128 to 127. 
- A call to `valueOf(...)` for a value inside that range returns the same cached object every time, rather than allocating a new one — identical in spirit to the string literal pool, just for small, commonly-used integer values, since these appear constantly in real code (loop counters, small flags, array indices) and sharing them saves meaningful allocation overhead.
- Outside that range, `valueOf(...)` allocates a genuinely new `Integer` object on every call — which is exactly why c == d is false above.
- `new Integer(int)` always creates a new Integer object, which is generally discouraged.
- The critical trap this creates — since javac rewrites Integer boxed = 100; into Integer.valueOf(100): code that happens to be tested only with small values (inside the cache range) can pass every test using `==` comparison, then silently break the moment a larger value flows through the exact same code path in production. 
- The fix, as with any object comparison, is always use `.equals()` for wrapper value comparison, never `==` — `==` on any object type (wrapper or otherwise) is always identity comparison, regardless of caching behavior; the cache just makes the bug intermittently "work by accident" for small values, which is worse than it failing consistently.

Other wrapper types have their own cache behaviors worth knowing precisely:
- `Boolean.valueOf(...)` — caches exactly two instances, `TRUE` and `FALSE` — every `Boolean` autobox is guaranteed to return one of these two shared objects, no exceptions.
- `Byte`, `Short`, `Long` — cache the full `-128` to `127` range (the complete value space for `Byte`, hence `Byte` is always fully cached).
- `Character` — caches `0` to `127` (all ASCII values).
- `Float`, `Double` — no caching at all, ever — floating-point equality/identity isn't given the same treatment, since distinct floating-point representations of "the same" value are a messier concept than exact integers.

### The classic NullPointerException trap — unboxing a null
```java
Integer boxed = null;
int unboxed = boxed;   // throws NullPointerException — compiler rewrites this to:
int unboxed = boxed.intValue();   // null.intValue() throws NPE
```
- This one compiles fine as there is no compilation error. We are assigning an unboxed value to a `boxed.intValue()`, the `intValue()` is operating on `null`. Hence, it throws a null pointer exception. 
- This is a common trap when using wrapper types and autoboxing/unboxing. Always ensure that the wrapper object is not null before unboxing it to a primitive type.

Solution:
```java
Integer count = getOptionalCount();   // might return null
if (count != null) {
    int c = count;                     // safe — only unboxes when non-null
}
```

### Mixed arithmetic — autoboxing interacting with operators
```java
Integer a = 10;
Integer b = 20;
Integer sum1 = a + b;    // BOTH operands are auto-unboxed to int, added as primitives, result reboxed
int sum2 = a + b;        // same as above, result is primitive int, no reboxing needed
```

- The `+` operator never works on Integer objects directly — there's no operator overloading in Java at all, wrapper classes included. 
- The compiler unboxes both operands to primitives, performs ordinary primitive arithmetic, then (if the result needs to be stored in a wrapper-typed variable) autoboxes the result back. 
- This has a real performance implication in a tight loop — every iteration potentially does unbox-compute-rebox, generating unnecessary object churn compared to just using the primitive type directly when boxing isn't actually needed for the variable's purpose.

### `==` between a primitive and a wrapper — a subtlety worth stating precisely
```java
Integer boxed = 1000;
int primitive = 1000;
System.out.println(boxed == primitive);   // true
// equivalent to boxed.intValue() == primitive
```

- When `==` compares a wrapper type against a primitive type, Java unboxes the wrapper and performs primitive comparison — this is different from wrapper-to-wrapper `==` (which is always identity). 
- The rule precisely: `==` between two reference types is identity; `==` where at least one side is a primitive forces unboxing and becomes a value comparison. 
- This single rule explains why the earlier cache-based surprises only happen in wrapper-to-wrapper comparisons, never in mixed wrapper-to-primitive ones.

## Operators with Wrapper Types

| Operator category                              | Example               | What happens                                                                                        |
|------------------------------------------------|-----------------------|-----------------------------------------------------------------------------------------------------|
| Arithmetic (`+ - * / %`)                       | `Integer a, b; a + b` | Both unboxed to primitives, computed, result reboxed if stored back into a wrapper                  |
| Relational (`< > <= >=`)                       | `a > b`               | Both unboxed to primitives, compared as primitives — result is a primitive `boolean`, never reboxed |
| Bitwise (`& \| ^` on numeric/Boolean wrappers) | `flagA & flagB`       | Both unboxed, computed as primitives                                                                |
| Logical (`&& \|\|` on `Boolean`)               | `boolA && boolB`      | Both unboxed to primitive `boolean`                                                                 |
| Compound assignment (`+=`, `-=`, etc.)         | `a += b`              | Both unboxed, computed, result reboxed back into the wrapper variable                               |
| Equality (`== !=`)                             | `a == b`              | No unboxing at all — pure reference/identity comparison, exactly like comparing any two objects     |


## Wrapper Class Traps
| Trap                                  | Rule                                                                                                            |
|---------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| `Integer == Integer` for small values | Appears to "work" due to the `-128..127` cache — not a guarantee for values outside that range                  |
| `Integer == Integer` for large values | `false` even for equal values — always genuinely distinct objects outside the cache                             |
| Fix for wrapper value comparison      | Always `.equals()`, never `==` — regardless of whether the value happens to fall in a cached range              |
| Unboxing a `null`                     | Throws `NullPointerException` at the point of the compiler-inserted `.intValue()`-style call, not at assignment |
| Wrapper arithmetic (`a + b`)          | Always unboxes to primitives, computes, then reboxes — no real operator exists on the wrapper type itself       |
| Mixed wrapper `==` primitive          | Forces unboxing — becomes value comparison, unlike wrapper-to-wrapper `==`                                      |


