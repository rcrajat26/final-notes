# Strings & the String Pool
## String internal structure
### Pre–Java 9 (every version up through 8):
```java
private final char[] value;   // the actual character data
private int hash;              // cached hashCode, lazily computed, 0 until first call
```

- Every character stored as a `char` — 2 bytes each, always, regardless of content — because char in Java is UTF-16 (fixed 16-bit).
- `value` is `final` — the array reference can never be repointed to a different array — and critically, String never exposes this array or any mutating method over it, so in practice the contents are never touched either. 
- The array being private and never handed out (not even a defensive copy is needed, since nothing ever gets a reference to it) is what makes the immutability guarantee airtight, not the final keyword alone — final only stops reassignment, not mutation of array elements; String's actual safety comes from simply never providing any method that writes into value.

### Java 9 onward — "Compact Strings" (JEP 254), a real, significant internal change:
```java
private final byte[] value;   // changed from char[] to byte[]
private final byte coder;      // 0 = LATIN1, 1 = UTF16
private int hash;
```

- The JVM inspects the actual content at construction time: if every character fits in LATIN-1 (i.e., the string is pure ASCII/Latin-1 range — the overwhelming majority of real-world strings, e.g. names, emails, account IDs in typical systems), it stores one byte per character instead of two — halving memory for most strings in a typical application, with zero API change visible to you as a caller. 
- If even a single character requires UTF-16 (non-Latin-1 content — e.g. strings containing certain non-English characters, emoji, etc.), the whole string falls back to the 2-bytes-per-char UTF-16 encoding, and coder is set accordingly. 
- No public API changed — length(), charAt(), etc. all still behave identically; the byte-packing/unpacking is handled internally and invisibly.
- While handling the byte value from Java 9 onward we have a supporting coder field. Its job is as below:
  - Latin-1: 1 byte represents 1 character/code unit. `value stored: LATIN1 → coder = 0`
  - UTF-16: typically 2 bytes represent 1 UTF-16 code unit. `value stored: UTF16 → coder = 1`

### Why `final` on the array isn't actually what makes `String` immutable
- `final char[] value;` only prevents value from being reassigned to point at a different array.
- It does **not** prevent the contents of the array from being changed (e.g., `value[0] = 'X';` would be legal if it were accessible).
- String's real immutability guarantee comes from a combination of:
  - value is private — no external code can reach it directly.
  - String never exposes any mutating method that would allow writing into value (no `setCharAt`, no `getValue()` returning the array, etc.).

## String API design — a few key points
- String is a reference type, not a primitive.
- String is `final` — cannot be subclassed, so no subclass can sneak in a mutating override of any method.

## Important Methods to Know (Grouped by Purpose)
### Inspection
| Method                            | What it does                                                               |
|-----------------------------------|----------------------------------------------------------------------------|
| `length()`                        | Number of characters                                                       |
| `isEmpty()`                       | `length() == 0`                                                            |
| `isBlank()`                       | (Java 11+) true if empty or all-whitespace                                 |
| `charAt(i)`                       | Character at index `i` — `StringIndexOutOfBoundsException` if out of range |
| `indexOf(...)`/`lastIndexOf(...)` | First/last position of a substring or char; `-1` if absent                 |
| `contains(sub)`                   | Whether a substring is present                                             |
| `startsWith(s)` / `endsWith(s)`   | Prefix/suffix check                                                        |

### Comparison
| Method                   | What it does                                    |
|--------------------------|-------------------------------------------------|
| `equals(o)`              | Content equality — case-sensitive               |
| `equalsIgnoreCase(s)`    | Content equality, ignoring case                 |
| `compareTo(s)`           | Lexicographic comparison (`Comparable<String>`) |
| `compareToIgnoreCase(s)` | Same, case-insensitive                          |

```java
String a = "Apple";
String b = "Banana";

System.out.println(a.compareTo(b)); // negative
System.out.println(b.compareTo(a)); // positive
System.out.println(a.compareTo("Apple")); // 0
// Strings are compared lexicographically (roughly dictionary order).
```

### Transformation
> Every single one of these returns a **NEW `String`**, per immutability.
 
| Method                           | What it does                                                                |
|----------------------------------|-----------------------------------------------------------------------------|
| `substring(begin[, end])`        | New string from `begin` (inclusive) to `end` (exclusive)                    |
| `concat(s)`                      | Appends `s` — equivalent to `+` for two strings                             |
| `replace(oldChar, newChar)`      | Replaces all occurrences (literal, not regex)                               |
| `replaceAll(regex, repl)`        | Regex-based replacement                                                     |
| `toUpperCase()`/`toLowerCase()`  | Case conversion (locale-sensitive — worth knowing for i18n edge cases)      |
| `trim()`                         | Removes leading/trailing ASCII whitespace                                   |
| `strip()`                        | (Java 11+) Unicode-aware whitespace trimming — replacement for `trim()`     |
| `split(regex)`                   | Splits into a `String[]` by a regex delimiter                               |
| `join(delimiter, elements...)`   | (Java 8+, static) Joins multiple strings with a delimiter                   |
| `repeat(n)`                      | (Java 11+) Repeats the string `n` times — version flag, didn't exist pre-11 |
| `format(...)` / `formatted(...)` | printf-style formatted string construction                                  |

### Conversion
| Method          | What it does                                                      |
|-----------------|-------------------------------------------------------------------|
| `toCharArray()` | Returns a new `char[]` copy of the content                        |
| `chars()`       | (Java 8+, returns an `IntStream`)                                 |
| `valueOf(x)`    | Static — converts almost any type, to its `String` representation |
| `getBytes()`    | Encodes to a `byte[]` using a charset                             |

### Identity / Pool-Related


| Method     | What it does                                           |
|------------|--------------------------------------------------------|
| `intern()` | Returns the canonical pooled instance for this content |

### Trap Summary
- `final char[] value` alone "is" immutability: **False** — `final` only blocks reassignment; the real guarantee is the array never being exposed or aliased anywhere in the API
- `char[]` vs `byte[]` internal storage: Changed in Java 9 (Compact Strings) — pure memory optimization, zero API/semantics change 
- `toCharArray()` mutation: Returns a copy — mutating it never affects the original `String`
- `replaceAll`: Takes a regex, not a literal — unescaped special characters misbehave
- `trim()` vs `strip()`: `trim()` = ASCII whitespace only, always existed; `strip()` = Unicode-aware, Java 11+ only
- `getBytes()` with no charset: Uses platform-default encoding — a portability bug waiting to happen; always specify the charset explicitly


## String is immutable
### what that actually guarantees, mechanically
- Every `String` object, once constructed, never changes its internal character data for its entire lifetime. 
- Every "modifying" operation (`concat`, `substring`, `replace`, `toUpperCase`, etc.) returns a brand-new `String` object — the original is untouched.

```java
String name = "Arjun";
name.toUpperCase();          // creates a NEW String "ARJUN" — discarded immediately, not assigned
System.out.println(name);    // still prints "Arjun" — trap for beginners
name = name.toUpperCase();   // must reassign to actually use the new object
```

### Why immutability was a deliberate design choice:
- String pool safety — pooled/shared `String` objects would be corrupted for every reference holding them if mutation were allowed.
- Thread safety — an immutable object can be freely shared across threads with zero synchronization needed, since there's nothing to race on.
- Hash code caching — since content can never change, `String` caches its `hashCode()` the first time it's computed (a private int hash field, computed lazily, left as 0 until first call) — safe only because the content is guaranteed never to change afterward.
- Security — class loading, file paths, network hosts, DB connection strings are frequently passed as `String`; immutability means once validated, that value can't be silently swapped out from underneath the code that validated it.

**More on, Hashcode problem with inspector string**:
- Any other class that mutates has this problem but we highlight string specifically
- This is because string, by far, is most widely used as a key in a HashMap or HashTable, along with that it gets used as JSON keys, IDs, cache keys. Hence the blast radius of this error on `String` is too wide, hence it gets a special mention.

## The String Pool (a.k.a. String Constant Pool)
### what it is, mechanically
- A special, dedicated memory region (historically part of the old PermGen; since Java 7, moved into the regular Heap) holding one shared copy of every distinct string literal the program uses.

```java
String a = "Arjun";   // literal — JVM checks pool first: not present, creates "Arjun" in pool, `a` points to it
String b = "Arjun";   // literal — JVM checks pool: "Arjun" already there, `b` points to the SAME object as `a`
System.out.println(a == b);   // true — same heap object, pool reuse

String c = new String("Arjun");   // bypasses pool lookup entirely — ALWAYS allocates a new heap object
System.out.println(a == c);        // false — different objects, even though content is identical
System.out.println(a.equals(c));   // true — content comparison (M11), correctly ignores identity
```

### `intern()` — manually forcing pool participation
```java
String c = new String("Arjun");
String d = c.intern();     // looks up "Arjun" in the pool; returns the pooled reference (creates it if absent)
System.out.println(a == d); // true — d now points to the same pooled object as a
```

- a == d → true (both point to P, the pooled object). 
- c == d → false. 
- c itself is completely untouched. intern() does not modify, relocate, pool, or destroy c in any way — it's a pure lookup-and-return method. c still exists on the heap, still a distinct, non-pooled object with content "Arjun", exactly as it was right after new String("Arjun") ran. Nothing about calling .intern() on it changes c's identity or its pool status.

**Compile-time constant folding — a sharp trap**
```java
String e = "Arj" + "un";              // BOTH operands are compile-time constants
System.out.println(a == e);            // true! — compiler folds this into the literal "Arjun" at COMPILE time, before pooling even runs

String f = "Arj";
String g = f + "un";                  // f is a variable, not a compile-time constant
System.out.println(a == g);            // false — g is computed at RUN time, so a new String object is created on the heap, not pooled
```
The rule precisely: 
- string concatenation of literals is evaluated by the compiler itself, producing a single literal in the bytecode — which then goes through the exact same pool lookup as any other literal. 
- Concatenation involving any non-constant variable happens at runtime, via a freshly constructed object, every single time — never pooled automatically.
- We can assume it this way the constant is known at the time of compilation the value inside a field can differ. However if the field was final, we can be sure that it won't change. 

```java
final String part2 = "Arj";            // final + literal initializer = compile-time constant
String g = part2 + "un";
System.out.println(a == g);            // true — 'final' + literal makes this foldable again
```

## StringBuilder/StringBuffer
- Since every String concatenation in a loop creates a brand-new object (immutability), repeated concatenation in a loop is expensive in cost:
```java
String result = "";
for (Account acc : accounts) {
    result = result + acc.getType() + ", ";   // new String object every single iteration — wasteful
}
```
- `StringBuilder` (not thread-safe, faster) / `StringBuffer` (synchronized, slower, legacy — kept for backward compatibility) hold a mutable internal char[] buffer that grows as needed, so repeated appends mutate one object instead of allocating a new one each time.

Internal structure
- Holds a char[] value (the buffer) and an int count (how much of the buffer is actually in use — the buffer's allocated capacity can be larger than the current content length).
- Default initial capacity: 16 characters, if constructed with no-arg new StringBuilder(). You can also specify an initial capacity (new StringBuilder(100)) or seed it with existing content (new StringBuilder("Arjun") — capacity becomes content length + 16).
- Growth strategy: when an append would exceed current capacity, the buffer is replaced with a new array, sized (roughly) (oldCapacity * 2) + 2 — doubling growth.
- This mechanism keeps it cheap. We need not create a new array each time. The existing array will have some buffer left. Hence this is not a hugely costly affair.

| Method                                   | What it does                                                        |
|------------------------------------------|---------------------------------------------------------------------|
| `append(x)`                              | Adds `x` (any type) to the end; returns `this`                      |
| `insert(index, x)`                       | Inserts `x` at a specific position, shifting existing content right |
| `delete(start, end)`                     | Removes characters in range `[start, end)`                          |
| `deleteCharAt(index)`                    | Removes a single character                                          |
| `replace(start, end, str)`               | Replaces a range with new content                                   |
| `reverse()`                              | Reverses the entire buffer in place                                 |
| `charAt(index)` / `setCharAt(index, ch)` | Read/write a single character directly                              |
| `length()`                               | Current content length (like `String.length()`)                     |
| `capacity()`                             | Current allocated buffer size (capacity can be larger than length)  |
| `toString()`                             | Produces an actual immutable `String` snapshot of current content   |

**When the compiler uses StringBuilder for you, silently**
- Worth knowing: a non-constant string concatenation like result + acc.getType() + ", " (where result is a loop variable, not a compile-time constant) is compiled by javac into roughly:
```java
result = new StringBuilder().append(result).append(acc.getType()).append(", ").toString();
```
- every single time through the loop, a brand-new StringBuilder is created, appended to, then converted back to a String and discarded.
- This is strictly better than manually doing naive String concatenation the long way, but still far worse than declaring one StringBuilder outside the loop and reusing it across all iterations:
```java
StringBuilder sb = new StringBuilder();
for (Account acc : accounts) {
    sb.append(acc.getType()).append(", ");
}
String result = sb.toString();   // one single String object created, at the very end
```

### `switch` on `String` — version note
- `switch` on `String` is supported since Java 7. Internally, the compiler generates a hash-based lookup table for the case labels, so performance is O(1) on average, not O(n) like a chain of if/else.


