# Diamond problem
- This shows up whenever there are two parent classes which are drawing from a common grandparent class.
- Consider classes `A`, `B`, `C`, and `D` as shown below
```
      A
     / \
    B   C
     \ /
      D
```

- `D` can reach a common ancestor `A` through `B` and `C`. 
- For the fields in `A`, will `D` get two copies, one through `B` and another through `C`? Or will it get one copy of fields of `A`? This is the diamond problem.
- Further if `B` and `C` override something, what will `D` receive? Which copy will `D` receive, `B`'s or `C`'s?
- If `B` and `C` change value independently then `D` starts seeing an ambiguous value each time.

## Solution in Java
- Java avoids the diamond problem by not allowing multiple inheritance of classes. A class can only extend one superclass.
- However, Java allows multiple inheritance of interfaces. With default methods in interfaces, the diamond problem can occur with behavior, **not state**.
- Java resolves this with the following rules:
  - A class's own method always wins over any interface default.
  - The most specific interface wins — if B extends A and both have greet(), B's version is used automatically (no ambiguity, no error).
  - If two unrelated interfaces both provide a default for the same method, it's a compile error — you must override and manually disambiguate with InterfaceName.super.method().

```java
interface A {
    default void greet() { System.out.println("Hello from A"); }
}

interface B extends A {
    default void greet() { System.out.println("Hello from B"); }
}

interface C extends A {
    default void greet() { System.out.println("Hello from C"); }
}

class D implements B, C {
    // COMPILE ERROR if omitted:
    // "class D inherits unrelated defaults for greet() from types B and C"
}
```

The compiler refuses to guess. D is forced to override greet() explicitly and decide what to do — including optionally delegating to one of the parents via InterfaceName.super.methodName():

```java
class D implements B, C {
    @Override
    public void greet() {
        B.super.greet();   // explicitly pick B's version
        C.super.greet();   // or call both
        System.out.println("Hello from D");
    }
}
```

- One subtlety worth noting — if B and C don't override greet() and both just inherit it from A, there's no conflict at all