---
Revisit later 
---

# G9. Performance & Cost Model

## 1. The mindset: measure first, then reason about cost

Performance work has two halves:

1. **A cost model.** A rough sense of which operations are cheap, which are expensive, and why. This lets you avoid obvious mistakes while writing code.
2. **Measurement.** Everything beyond the obvious must be profiled, because the JVM optimizes aggressively and intuition is often wrong.

> "Premature optimization is the root of all evil" (Knuth) really means: don't make code harder to read for a speedup you haven't measured on a path that is actually hot.

In a typical backend service such as payments processing, the time is rarely spent in CPU-bound Java code. It goes to **I/O**: database round trips, network calls, serialization and parsing. Fixing one N+1 query pattern beats any amount of loop tuning. Use micro-level knowledge for hot loops and library or infrastructure code, and think in terms of round trips everywhere else.

---

## 2. Orders of magnitude (the real cost model)

Numbers are rough and hardware-dependent. Only the **ratios** matter.

| Operation | Rough cost |
|---|---|
| Primitive arithmetic, local variable access | ~1 ns or less (often free after JIT) |
| Field access, getter call (inlined by JIT) | ~1 ns |
| Object allocation (fast path) | ~10–20 ns, plus GC cost later |
| Main-memory access (CPU cache miss) | ~100 ns |
| Exception creation (stack trace filled) | ~1–50 µs, grows with stack depth |
| `String.format(...)` | ~1 µs |
| `System.out.println` in a hot loop | µs to ms (synchronized, I/O-bound) |
| Reading from SSD | ~100 µs |
| Network round trip, same data centre | ~0.5 ms |
| Database query | ~1–50 ms |
| Network round trip, cross-region | ~100 ms |

Each step down is roughly 10x to 1000x slower. One avoidable DB call costs as much as millions of field accesses. That is why the I/O-level fixes dominate.

---

## 3. Memory cost: what an object actually occupies

Memory matters because it drives GC work and CPU-cache efficiency. On 64-bit HotSpot (Java 21 default, compressed references on, heap below ~32 GB):

- Every object has a **12-byte header** (mark word plus class pointer).
- References are **4 bytes** (compressed oops). Without compression they are 8 bytes.
- Object size is **rounded up to a multiple of 8 bytes**.
- Arrays add a 4-byte length, giving a **16-byte header**.
- Newer JDKs (24+) can shrink the header to 8 bytes (compact object headers). That is not the Java 21 baseline.

```
Account { String accountId; AccountType type; BigDecimal balance; }

header        12 bytes
accountId ref  4
type ref       4
balance ref    4
              ---
              24 bytes  (this is only the Account object; the String and BigDecimal are separate objects)
```

| Item | Approx size |
|---|---|
| `Integer` | 16 bytes (12 header + 4 value) |
| `Long` / `Double` | 24 bytes (12 + 4 padding + 8) |
| `int[1_000_000]` | ~4 MB |
| `Integer[1_000_000]` (values outside the small-value cache) | ~4 MB of references **+ 16 MB** of `Integer` objects, about **5x** more |
| `String "hello"` | 24 bytes for the `String` object + 24 bytes for its `byte[]` = ~48 bytes |

Three consequences:

- **Wrappers cost memory and indirection.** A primitive array is one contiguous block. An array of wrappers is an array of pointers to scattered objects.
- **Many small objects add up** because each pays for a header and for padding.
- **Object graphs are pointer-heavy.** Following a pointer can mean a cache miss (~100 ns), while scanning a primitive array is nearly free per element because the CPU prefetches sequentially.

To measure real sizes instead of guessing, use the OpenJDK **JOL** (Java Object Layout) tool.

---

## 4. Allocation and garbage collection

### Allocation is cheap

Each thread gets a private chunk of the young generation, called a **TLAB** (thread-local allocation buffer). `new` is usually a **pointer bump** inside it, with no locking and no search for free space. Memory comes back zeroed.

### GC cost depends on live objects, not dead ones

Most objects die young. The **young generation** is collected by copying the survivors elsewhere. The dead objects aren't visited at all. So:

| Object lifetime | Cost |
|---|---|
| Dies almost immediately (temporary strings, small DTOs, iterators) | Nearly free |
| Lives a long time in the old generation (caches, loaded config) | Not collected often, but a large live set makes full or concurrent cycles longer |
| Lives "a bit too long" (survives a few young collections, then dies) | **Most expensive**: copied around, then promoted to the old generation, then garbage there |

GC pauses ("stop-the-world") are the visible cost. The default collector is **G1** (since Java 9). **ZGC** (generational mode added in Java 21) aims at very short pauses. Collector tuning is operational and out of scope here. The design lesson is: **don't create unnecessary long-lived garbage, and don't fear short-lived allocation.**

### Escape analysis

The JIT can prove that an object **never escapes** the method (it isn't stored in a field, returned or passed out). It can then skip the heap allocation entirely (**scalar replacement**) and keep the fields in registers. This means:

- Small temporary objects (a short-lived `Point`, an iterator, a builder used inside one method) can cost **nothing** after JIT.
- **Hand-written object pools for small objects are usually counterproductive**: they add complexity, keep objects alive longer (promotion cost) and block escape analysis. Pooling is justified only for genuinely expensive resources (connections, big buffers).

### Immutability is rarely the bottleneck

Defensive copies and "wither" methods allocate, but short-lived allocation is cheap. Choose immutability for correctness and treat the allocation cost as negligible until a profiler says otherwise.

---

## 5. Common code-level costs

### a) Autoboxing in loops

```java
Long total = 0L;                          // wrapper
for (long i = 0; i < 10_000_000; i++) {
    total += i;                           // unbox, add, box a NEW Long each iteration (outside -128..127)
}

long total2 = 0L;                         // primitive: no allocation at all
```

The first form often allocates millions of `Long` objects. Escape analysis may remove some, but don't count on it. Use primitives for accumulators, counters and loop variables, and wrappers only where `null` or an object is genuinely required.

### b) String concatenation

```java
String s = "";
for (Account a : accounts) s += a.getId() + ",";     // O(n²): new String each iteration

StringBuilder sb = new StringBuilder();              // O(n)
for (Account a : accounts) sb.append(a.getId()).append(',');
```

- Concatenating in a **loop** copies the growing string every iteration, so the cost is quadratic.
- A **single** expression (`"id=" + id + ", type=" + type`) is fine and needs no builder.
- Version note, as a refinement of the usual explanation: before Java 9 the compiler rewrote a concatenation expression into `StringBuilder` calls. **From Java 9 (JEP 280) it emits an `invokedynamic` call** and the runtime picks the strategy. The loop problem is the same either way.
- `String.format` is much slower than `+` or a builder (it parses the format string each time). Fine for occasional use, bad inside hot paths.

### c) Regex and string utilities

`String.matches`, `replaceAll` and multi-character `split` all **compile a regex pattern on every call**. In a hot path, compile once and reuse:

```java
private static final Pattern ACCOUNT_ID = Pattern.compile("[A-Z]{2}-\\d{6}");   // compiled once
boolean ok = ACCOUNT_ID.matcher(id).matches();
```

`String.substring` **copies** its characters (since Java 7u6). It is not a cheap view. Use `replace(CharSequence, CharSequence)` instead of `replaceAll` when no regex is needed.

### d) Exceptions

The expensive part of an exception is not `throw` or `catch`. It is **`fillInStackTrace`**, which walks and records the whole call stack when the `Throwable` is constructed. Cost grows with stack depth, and in a framework app the stack is often 100+ frames deep.

- **Don't use exceptions for normal control flow.** Parsing a field with `try { Integer.parseInt(s) } catch (NumberFormatException e)` is fine when failure is rare, and very slow if most inputs are invalid. Check with a cheap test first.
- For rare hot-path cases there is a constructor that **disables the stack trace**:

```java
class FastFail extends RuntimeException {
    FastFail(String msg) { super(msg, null, false, false); }   // no suppression, no stack trace
}
```

Use it only for an exception meant as a signal, because you lose all diagnostic information.

### e) Reflection

Looking up a member (`getDeclaredMethod`) is expensive, and `invoke` boxes arguments and checks access. **Look up once, cache the `Method`/`Field`, reuse it.** Since Java 18 (JEP 416) core reflection is implemented on top of method handles, which narrows the gap but doesn't remove it.

### f) Logging

```java
log.debug("Account state: " + account.expensiveDump());       // dump is built even if DEBUG is off
log.debug("Account state: {}", account);                      // string built only if DEBUG is enabled
```

The argument expression is evaluated **before** the method is called. If it is costly, use parameterized messages (or an `if (log.isDebugEnabled())` guard) so the work is skipped when the level is disabled. Also avoid `System.out.println` in production hot paths.

### g) Money and numbers

`BigDecimal` is slower and allocates more than `double`. For money it is still the right choice. Correctness beats a speedup, and arithmetic is rarely the bottleneck compared with I/O. For counters and indices use `int`/`long`.

---

## 6. Data layout and locality

```java
int[][] grid = new int[4000][4000];

// Fast: walks memory in the order it is laid out
for (int r = 0; r < 4000; r++)
    for (int c = 0; c < 4000; c++) sum += grid[r][c];

// Slower, often by several times: jumps between row arrays on every step
for (int c = 0; c < 4000; c++)
    for (int r = 0; r < 4000; r++) sum += grid[r][c];
```

A Java 2D array is an **array of references to row arrays**. Each row is contiguous, but rows aren't. Iterating along a row uses the CPU cache and its prefetcher well. Iterating down a column causes a cache miss almost every step. The same idea applies to objects: a contiguous `double[]` of balances scans far faster than a graph of objects, each pointing to a separate `BigDecimal`. The program's logic is identical, but the memory access pattern differs.

Algorithmic cost still outranks all of this. Scanning 100,000 accounts for each of 100,000 transactions to find a match is 10 billion comparisons. A keyed lookup (the structures belong to the collections material) turns it into 100,000. **Fix the algorithm before touching the micro-level.**

---

## 7. The JIT and warm-up

The JVM starts by **interpreting** bytecode. It counts how often methods and loops run, and compiles the hot ones to native code using tiered compilation (a quick C1 compile, then an aggressively optimized C2 compile for the hottest code). The optimizations include inlining, escape analysis and devirtualization. Consequences:

- **Early calls are slower** than steady state. The first requests after a deploy run on interpreted or lightly compiled code. This is "warm-up".
- **Short-lived processes** (CLI tools, cold Lambda invocations) may never reach the optimized tiers. This is the reason for ahead-of-time options such as GraalVM native image, which trades peak throughput and runtime flexibility for fast startup and no warm-up.
- **Tiny methods and getters cost nothing** once inlined. Don't remove accessors "for speed".
- **Polymorphic calls:** if a call site only ever sees one or two concrete classes, the JIT inlines it. A site that sees many classes (megamorphic) falls back to a vtable lookup. That is slower, but rarely a problem in practice.

How the JIT and bytecode work is covered properly in a later module. For now, remember that **measured speed depends on JIT state**, which is what makes benchmarking tricky.

---

## 8. Measuring correctly

### Why a naive timing loop lies

```java
long t0 = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) compute(i);       // result unused
long t1 = System.nanoTime();
System.out.println((t1 - t0) / 1e6 + " ms");
```

This is unreliable for several reasons:

| Problem | Effect |
|---|---|
| **Warm-up** | You are timing a mix of interpreted and compiled code |
| **Dead-code elimination** | If the result is unused, the JIT may delete the work entirely, so you measure nothing |
| **Constant folding** | A fixed input can be computed at compile time |
| **Loop optimizations** (e.g. on-stack replacement) | The loop is compiled differently than the same code in a real call path |
| **GC and noise** | One run on a loaded machine proves nothing |

### Microbenchmarks: use JMH

**JMH** (Java Microbenchmark Harness, from the OpenJDK team) handles forking, warm-up, repeated measurement and dead-code defeat:

```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class ConcatBenchmark {
    String a = "ACC", b = "-123456";

    @Benchmark
    public String plus() { return a + b; }           // returning the value prevents dead-code elimination
}
```

Even with JMH, a microbenchmark answers only "how fast is this isolated operation", which may not matter for your application.

### Application-level profiling

| Tool | Use |
|---|---|
| **JFR (Java Flight Recorder)** | Low-overhead profiling built into the JDK (free since 11): CPU hot spots, allocations, GC, locks, I/O |
| **async-profiler** | Sampling CPU and allocation flame graphs |
| **GC logs** (`-Xlog:gc*`) | Pause times, allocation rate, heap growth |
| **`jcmd`, VisualVM** | Heap histograms, thread dumps, live inspection |
| **JOL** | Exact object sizes |
| **APM / tracing** (distributed tracing, DB query timing) | Which **call** is slow in a distributed request. This usually finds the real bottleneck. |

Process: measure with realistic data and load, find the biggest contributor, fix **that**, measure again.

---

## 9. Memory leaks in a GC language

Java doesn't leak by forgetting to free. It leaks by **keeping objects reachable that you no longer need**:

- A `static` collection that only grows (a cache with no eviction).
- Listeners or callbacks registered and never removed.
- Long-lived objects holding references to short-lived ones (an inner-class instance pinning its outer object).
- Classes loaded by a custom class loader that stay reachable after a redeploy.
- Unbounded caches, or session data that is never cleared.

Symptoms are a steadily rising old-generation size and longer GC cycles. A heap dump analysed with a tool such as Eclipse MAT shows which reference chain is holding the objects.

---

## 10. Myths and what actually matters

| Folk wisdom | Reality |
|---|---|
| "`final` methods and classes are faster" | Not with a modern JIT, which already knows what is overridden. Use `final` for design, not speed. |
| "Getters/setters are slower than field access" | They are inlined away. |
| "Use `StringBuilder` for every concatenation" | Only in loops or when assembling in pieces. A single expression is fine. |
| "`i++` vs `++i`, manual loop unrolling" | The JIT handles it. |
| "Object creation is expensive, pool everything" | Allocation is cheap. Pool only expensive resources. |
| "Avoid interfaces and abstraction for speed" | Monomorphic and bimorphic calls are inlined. |
| "Catching exceptions is slow" | **Creating** them (stack trace) is the costly part. |
| "Primitives vs wrappers doesn't matter" | It matters in hot paths and large arrays (section 3). |

What does matter, in order of typical payoff:

1. Avoid unnecessary **I/O round trips** (N+1 queries, chatty calls, missing batching).
2. Use the right **algorithm and data access pattern**.
3. Avoid wasteful **allocation and boxing** in hot loops.
4. Reuse **expensive objects** (compiled patterns, formatters, HTTP clients, connection pools).
5. Mind **memory footprint and locality**.
6. Last: micro-tuning, and only after profiling.

---

## 11. Trap summary

1. Every `Integer` costs 16 bytes, so `Integer[]` can use ~5x the memory of `int[]`.
2. `Long total += x` in a loop allocates a new `Long` per iteration. Use `long`.
3. String `+=` in a loop is O(n²). Use `StringBuilder`.
4. From Java 9 a single concatenation expression is compiled via `invokedynamic`, not `StringBuilder` calls.
5. `matches`, `replaceAll` and regex `split` recompile the pattern on every call. Precompile with `Pattern`.
6. `substring` copies.
7. Exception cost is in building the stack trace, not in `throw`/`catch`. Don't use exceptions as ordinary branching.
8. Cache reflective `Method`/`Field` lookups.
9. Log arguments are evaluated before the call. Use `{}` placeholders or a level guard.
10. Allocation is cheap, but data that lives "a little too long" is the costly kind for the GC.
11. Don't pool small objects. It hurts escape analysis and promotes garbage.
12. A naive `nanoTime` loop doesn't measure what you think: use JMH, and only trust application-level profiles for real decisions.
13. The JVM is slow at the start (interpreter), so first-request latency isn't steady-state performance.
14. Java leaks via **reachability** (static collections, listeners, class loaders), not by forgetting to free.
15. Fix the algorithm and the I/O first, since they outweigh any micro-optimization.

---

Tell me if you want any of this expanded. Likely candidates are a runnable JMH comparison of boxing, string-building and exception cost, a JOL walk-through of the `Client`/`Account` object graph with exact byte sizes, or a step-by-step reading of a GC log and a JFR recording. When you're ready, G10 (JVM & Bytecode Essentials, including the JIT and GraalVM) is next.