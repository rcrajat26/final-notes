# Lexical foundations 
## Identifiers
- Identifier can have letters, digits, `_`, and `$`, And can't start with a digit. `_` alone is not a valid variable.
- Usually `$` is used for internal variables, machine-generated/compiler-generated variables

## Keywords 
- Keywords are reserved words that have a predefined meaning in Java. They cannot be used as identifiers (variable names, method names, class names, etc.).
- **Contextual keywords** — words that act like keywords only in certain positions, but are legal identifiers elsewhere: var, yield, record, sealed. Unlike true keywords (class, public, if), these won't break old code that happened to use them as variable names.

## Literals 
### Integer literals
```java
int dec = 100;
int hex  = 0x64;   // hex
int oct  = 0144;   // octal — leading 0
int bin  = 0b1100100; // binary
```
> Trap: int x = 010; is 8, not 10 — leading zero means octal. Easy to write by accident (e.g. zero-padding something) and get silently wrong math.

- **Underscore for readability**: `int million = 1_000_000;` is the same as `1000000`. Underscores can be placed anywhere between digits, but not at the start or end, and not next to a decimal point or right before an `L`/`f`/`d` suffix.

### Floating-point literals 
```java
double d = 3.14;
float f  = 3.14f;   // 'f' suffix required
```

> Trap: float f = 1.1; does not compile. A bare decimal literal is always double by default — you must write 1.1f

### Character and string literals 
```java
char c = 'A';
String s = "hello";
```

> null is a literal too — it has its own special "null type," not tied to any class.

### Boolean literals 
```java
boolean t = true;
boolean f = false;
```

### Enhanced for loop 
```java
for (String s : listOfStrings) {
    System.out.println(s);
}
```

Internally: for an array, this desugars to an index-based loop. For anything implementing Iterable, it desugars to calling .iterator() and looping with hasNext()/next(). Worth knowing conceptually — you're not writing anything new, just syntax sugar over what you'd do manually.


### switch (classic form)
```java
switch (day) {
    case 1:
        System.out.println("Mon");
        break;
    case 2:
        System.out.println("Tue");
        break;
    default:
        System.out.println("Unknown");
}
```

- Permitted switch selector types: byte, short, char, int (and their wrapper classes), String, and enum. Nothing else (no long, no boolean, no arbitrary objects) in classic switch.
- Switching on a String or wrapper type with a null value throws NullPointerException at runtime — the switch statement doesn't guard against it.

> Switch expressions (Java 14+) are to be covered later.

### Assert
```java
assert x > 0 : "x must be positive";
```
- If the boolean expression is false, an AssertionError is thrown with the given message.
- It's an Error, meaning "this is a programming bug, not a recoverable condition"
- Assertions are disabled by default at runtime. To enable them, run the JVM with the `-ea` (or `-enableassertions`) flag.
- Assertions are meant for catching your own bugs during development/testing — invariants that should never be false if your code is correct. They're not meant for validating user input or handling expected failure conditions
- Assertion error can be caught in the exception try-catch block as `Error` extends `Throwable` But this is not recommended to be used 

