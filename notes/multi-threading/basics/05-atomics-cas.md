# Atomics and compare-and-swap
## Compare-and-swapp
- Every modern CPU has an instruction that does this atomically, in hardware:
```
Look at given memory location. If it currently holds value X, replace it with value Y and report success. If it holds anything else, change nothing and report failure.
```

- The whole read-compare-write is indivisible — the hardware guarantees no other core can modify that address in the middle.

In Java it surfaces as:
```java
boolean compareAndSet(long expectedValue, long newValue)
```

In other words, CAS is a single atomic CPU instruction that does this pseudocode atomically:
```
boolean compareAndSwap(address, expectedValue, newValue) {
    if (*address == expectedValue) {
        *address = newValue;
        return true;
    }
    return false;
}
```

- This directly makes a hardware instruction call something like `LOCK CMPXCHG` machine instruction.
```java
public class AtomicInteger {
    private volatile int value;

    public final boolean compareAndSet(int expect, int update) {
        return U.compareAndSetInt(this, VALUE_OFFSET, expect, update);
    }
}
```

- The implementation of AtomicInteger shows when we call ai.compareAndSet() internally it calls Unsafe.compareAndSetInt() which is a native method that calls the underlying CPU instruction to perform the compare-and-swap operation atomically.
- `VALUE_OFFSET` is the byte offset of `value` within the object's memory layout, computed once via `Unsafe.objectFieldOffset`
- The `volatile` keyword ensures that reads and writes to `value` are not cached in registers or reordered, so that all threads see the most up-to-date value. Hence the AtomicInteger must be `volatile`

**What actually happens on the wire (cache coherence)**
Say two threads on two cores both call atomicInt.incrementAndGet():
- Core A's L1 cache holds the cache line containing value in Shared state (both cores have read it).
- Core A issues LOCK CMPXCHG. Hardware must upgrade that line to Exclusive — it sends an invalidation over the cache-coherence interconnect, forcing Core B's copy to go Invalid.
- Core A now has exclusive ownership, performs the compare + write atomically, line becomes Modified.
- Core B tries the same instruction — cache miss (line was invalidated), must fetch the now-Modified line from Core A (or via shared LLC), then retry.*


## Why this changes things

Look again at the lost update:

| # | thread A | thread B | heap |
|---|---|---|---|
| 1 | reads 50000 | | 50000 |
| 2 | | reads 50000 | 50000 |
| 3 | computes 51000 | | 50000 |
| 4 | | computes 52000 | 50000 |
| 5 | writes 51000 | | 51000 |
| 6 | | writes 52000 | **52000** |

Step 6 is where the damage happens: B writes unconditionally, having no idea the value changed since it read. Compare-and-swap makes step 6 **conditional**. B would say "write 52000, but only if the value is still 50000" — the hardware sees 51000, refuses, and tells B it failed. B then re-reads and tries again, this time computing 53000.

- Since CAS can fail (someone else won the race), Java code built on it always loops:
```java
public final int incrementAndGet() {
    int prev, next;
    do {
        prev = get();
        next = prev + 1;
    } while (!compareAndSet(prev, next));  // retry on failure
    return next;
} 
```
- This is optimistic concurrency — no thread ever blocks/parks; a losing thread just retries with a fresh read. Under contention this beats a mutex because there's no OS-level scheduling involved, just a spin

### Example:
```java
class Wallet {
    private final AtomicLong balance = new AtomicLong();

    boolean withdraw(long amt) {
        while (true) {
            long current = balance.get();
            if (current < amt) return false;              // check
            if (balance.compareAndSet(current, current - amt)) {
                return true;                              // act, only if still unchanged
            }
            // somebody changed it between our read and our write — re-read and redo
        }
    }
}
```
- The check is on a value, and the write only succeeds if that value is still in place.
- f another thread withdrew in between, the CAS fails and the check runs again with fresh data. The balance can never go negative.

**The limit: it only works for one variable**
```java
available.addAndGet(-amt);    // atomic
reserved.addAndGet(amt);      // atomic
// the pair is still not atomic
```
- Between those two lines the invariant available + reserved is false.
- CAS is atomic for one memory location. Two locations are two operations.

**The fix: make two fields into one**
```java
record Balance(long available, long reserved) { }

class Wallet {
    private final AtomicReference<Balance> state;

    Wallet(long opening) { state = new AtomicReference<>(new Balance(opening, 0)); }

    boolean reserve(long amt) {
        while (true) {
            Balance cur = state.get();
            if (cur.available() < amt) return false;
            Balance next = new Balance(cur.available() - amt, cur.reserved() + amt);
            if (state.compareAndSet(cur, next)) return true;
        }
    }
}
```
- Two fields now move together because they are one reference, and a reference write is a single atomic operation. 
- The old Balance is never mutated, so any thread holding one has a consistent, immutable snapshot that will never change under it.
- This pattern — immutable state object, swapped by CAS — is the standard lock-free answer to multi-field invariants, and it works for any number of fields

| Contention | Better choice |
|---|---|
| Low to moderate | CAS — no blocking, no context switches |
| Very high, single hot variable | `LongAdder` (§7), or rethink the design |
| Long critical sections | a lock — CAS retries would waste too much work |
| Anything involving I/O | a lock, and held only around the arithmetic (file 04 §5) |

## All relevant classes
Here's the full picture of `java.util.concurrent.atomic`, organized by what they're actually for.

## Single-value atomics

| Class | Wraps |
|---|---|
| `AtomicBoolean` | `boolean` |
| `AtomicInteger` | `int` |
| `AtomicLong` | `long` |
| `AtomicReference<V>` | any object reference `V` |

These are the "workhorses" — all support `get/set`, `compareAndSet`, `getAndSet`, `getAndUpdate`/`updateAndGet` (functional-style CAS loop), `getAndAccumulate`/`accumulateAndGet`, and (for Integer/Long) `getAndIncrement`, `getAndAdd`, etc.

Note: there's **no `AtomicFloat`/`AtomicDouble`** — the JDK considers these rare enough that you're expected to bit-cast via `Float.floatToIntBits`/`Double.doubleToLongBits` into an `AtomicInteger`/`AtomicLong` if you truly need it. `LongAdder`/`DoubleAdder` (below) partially fill that gap for the "adder" use case.

## ABA-safe variants

| Class | Purpose |
|---|---|
| `AtomicStampedReference<V>` | Reference + integer version stamp, both compared atomically — solves ABA by detecting "value changed and changed back" |
| `AtomicMarkableReference<V>` | Reference + single boolean mark (lighter-weight than a full stamp) — common for "logically deleted" flags in lock-free structures (e.g., marking a node as deleted before physically unlinking it) |

## Array variants

| Class | Purpose |
|---|---|
| `AtomicIntegerArray` | Atomic element-wise access to an `int[]` |
| `AtomicLongArray` | Atomic element-wise access to a `long[]` |
| `AtomicReferenceArray<V>` | Atomic element-wise access to an object `V[]` |

Same operations as the scalar versions, but indexed — `get(i)`, `compareAndSet(i, expected, new)`, etc. Useful for lock-free structures like hash tables or striped counters where you need CAS per-slot rather than one global atomic.

## Field updaters (atomic access to a *plain* field of an existing class, without changing its type)

| Class | Purpose |
|---|---|
| `AtomicIntegerFieldUpdater<T>` | Reflection-based atomic CAS on a `volatile int` field of class `T`, without wrapping it in an `AtomicInteger` object |
| `AtomicLongFieldUpdater<T>` | Same, for `volatile long` |
| `AtomicReferenceFieldUpdater<T,V>` | Same, for a `volatile` reference field |

Why these exist: if you have millions of instances of a class and each needs one atomically-updatable field, wrapping each field in its own `AtomicInteger` object costs an extra object allocation per instance. A `FieldUpdater` is a **single shared object** (created once, statically) that does `Unsafe`-style CAS directly on the raw field of whichever instance you pass in — no per-instance wrapper needed.

```java
class Task {
    volatile int state;
    static final AtomicIntegerFieldUpdater<Task> STATE_UPDATER =
        AtomicIntegerFieldUpdater.newUpdater(Task.class, "state");
}
// usage: Task.STATE_UPDATER.compareAndSet(someTaskInstance, 0, 1);
```

The field must be declared `volatile` and accessible from the calling context (no private-across-modules issues) — the updater does reflective/`Unsafe`-based access under the hood, and will throw at construction time if the field doesn't qualify.

These are largely **superseded by `VarHandle`** (JDK 9+) for new code, but remain in the JDK for back-compat and are still used internally in a lot of pre-9 concurrency code (e.g., `AbstractQueuedSynchronizer` historically used one for its `state` field before the `VarHandle` migration).

## High-throughput accumulators (`java.util.concurrent.atomic`, added JDK 8)

| Class | Purpose |
|---|---|
| `LongAdder` | A striped/sharded counter — multiple internal `Cell`s (each a padded `long`) that different threads write to, summed only when you call `.sum()`. Trades exact-time consistency for dramatically better write throughput under high contention (avoids the cache-line ping-pong we discussed with a single `AtomicLong`) |
| `DoubleAdder` | Same idea, for `double`, using the same striping trick with `Double.doubleToRawLongBits`/back |
| `LongAccumulator` | Generalizes `LongAdder` to an arbitrary associative binary function (not just `+`), e.g. `max`, via `LongBinaryOperator` |
| `DoubleAccumulator` | Same, for `double` |

These all descend from an internal abstract base, `Striped64`, which is the actual mechanism (per-thread/per-core hashed "Cells" instead of one contended field). Use `LongAdder` instead of `AtomicLong` whenever you only care about the *aggregate* count and many threads are incrementing concurrently — `Metrics`/counters are the classic use case, not "I need to read the exact current value on every single write."

## Where the *implementation* mechanism lives (not `Atomic*` classes themselves, but what backs them)

| Class | Role |
|---|---|
| `java.lang.invoke.VarHandle` | JDK 9+ general-purpose mechanism for typed, low-level atomic/volatile access to fields, array elements, or off-heap memory — obtained via `MethodHandles.lookup().findVarHandle(...)` or similar. This is the modern replacement for both `Unsafe` and `FieldUpdater` in new code you write yourself. (As we found earlier, `AtomicInteger`/`AtomicLong` *don't* actually use this internally due to bootstrap ordering, but conceptually this is the "correct" modern building block, and things like `AtomicReference` for *your own* classes' fields would typically be written with `VarHandle` today.) |
| `sun.misc.Unsafe` / `jdk.internal.misc.Unsafe` | The original low-level intrinsic-backed mechanism (`compareAndSwapInt`, `getAndAddLong`, etc.) — not public API, used internally by the JDK, increasingly restricted/hidden as the module system tightens |

## Quick decision guide

- Single counter/flag/reference, low-to-moderate contention → `AtomicInteger`/`AtomicLong`/`AtomicBoolean`/`AtomicReference`
- Lock-free stack/queue/linked structure needing to detect "changed and changed back" → `AtomicStampedReference` (or `AtomicMarkableReference` if you just need a delete-flag)
- Array of independently-CAS'able slots → `AtomicIntegerArray`/`AtomicLongArray`/`AtomicReferenceArray`
- One field on a huge number of plain objects, avoiding per-object wrapper allocation → `*FieldUpdater` (legacy) or `VarHandle` (modern)
- High-write-throughput counter where you rarely read, and many threads → `LongAdder`/`DoubleAdder`
- Same but with custom reduction (max/min/etc.) → `LongAccumulator`/`DoubleAccumulator`

## Methods
Here's the consolidated method reference across all `Atomic*` classes.

## Core read/write methods

| Method | Usage | Available in |
|---|---|---|
| `get()` | Reads current value (volatile semantics) | `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference<V>`, Array classes (`get(int i)`) |
| `set(newValue)` | Writes value (volatile semantics — visible to all threads immediately) | Same as above (Array: `set(int i, val)`) |
| `lazySet(newValue)` | Writes value with weaker (release-only) ordering — faster, but other threads may briefly see the stale value | Same as above |
| `getPlain()` / `setPlain(v)` | Read/write with **no** volatile guarantees at all (plain field semantics) — JDK 9+ | `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference` |
| `getOpaque()` / `setOpaque(v)` | Weaker-than-volatile but stronger-than-plain ordering (no reordering of *this* variable's accesses, but no broader fence) — JDK 9+ | Same |
| `getAcquire()` / `setRelease(v)` | Acquire/release semantics (like `synchronized` lock/unlock boundaries) — JDK 9+ | Same |

## Compare-and-swap family

| Method | Usage | Available in |
|---|---|---|
| `compareAndSet(expected, new)` | The core CAS: swap only if current value equals `expected`; returns `boolean` | All scalar, array (`(int i, expected, new)`), stamped/markable (with extra stamp/mark args), FieldUpdater (`(T obj, expected, new)`) |
| `weakCompareAndSet(expected, new)` | **Deprecated since JDK 9** — semantically confusing name; use `weakCompareAndSetPlain` instead | All scalar/array classes |
| `weakCompareAndSetPlain/Acquire/Release/Volatile(expected, new)` | "Weak" CAS variants — JVM allowed to spuriously fail even if values match, in exchange for possibly cheaper hardware instructions (useful in retry loops where a spurious failure just means "try again") — JDK 9+ | `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference` |
| `compareAndExchange(expected, new)` | Like `compareAndSet` but returns the **witness value** (what was actually there) instead of a boolean — lets you retry without a second `get()` — JDK 9+ | `AtomicInteger`, `AtomicLong`, `AtomicReference` |
| `compareAndExchangeAcquire/Release(expected, new)` | Same, with weaker memory ordering | Same |

## Get-and-set / functional update

| Method | Usage | Available in |
|---|---|---|
| `getAndSet(newValue)` | Atomically swaps in `newValue`, returns the **old** value | All scalar, array (`(int i, val)`), FieldUpdater |
| `getAndUpdate(UnaryOperator)` | Applies a function to current value, stores result, returns **old** value (internally a CAS retry loop) | `AtomicInteger` (`IntUnaryOperator`), `AtomicLong` (`LongUnaryOperator`), `AtomicReference` (`UnaryOperator<V>`) — **not** `AtomicBoolean` |
| `updateAndGet(UnaryOperator)` | Same, but returns the **new** value | Same set as above |
| `getAndAccumulate(x, BinaryOperator)` | Combines current value with `x` via a function, stores result, returns **old** value | `AtomicInteger` (`IntBinaryOperator`), `AtomicLong` (`LongBinaryOperator`), `AtomicReference` (`BinaryOperator<V>`) |
| `accumulateAndGet(x, BinaryOperator)` | Same, returns **new** value | Same |

## Numeric increment/add (Integer/Long only)

| Method | Usage | Available in |
|---|---|---|
| `getAndIncrement()` / `incrementAndGet()` | `+1`, returns old/new value respectively | `AtomicInteger`, `AtomicLong`, and array/FieldUpdater equivalents (indexed/object-qualified) |
| `getAndDecrement()` / `decrementAndGet()` | `-1`, returns old/new value | Same |
| `getAndAdd(delta)` / `addAndGet(delta)` | Adds `delta`, returns old/new value | Same |

## `Number` conversions (Integer/Long extend `Number`)

| Method | Usage | Available in |
|---|---|---|
| `intValue()` / `longValue()` / `floatValue()` / `doubleValue()` | Widening/narrowing conversions of the current value | `AtomicInteger`, `AtomicLong`, and (differently) `LongAdder`/`DoubleAdder`/`LongAccumulator`/`DoubleAccumulator` |

## Array-specific (`AtomicIntegerArray`, `AtomicLongArray`, `AtomicReferenceArray<V>`)

| Method | Usage |
|---|---|
| `length()` | Returns array length |
| All methods above, indexed | Every method takes a leading `int i` index parameter, e.g. `compareAndSet(int i, expected, new)`, `incrementAndGet(int i)` |

## Field-updater specific (`AtomicIntegerFieldUpdater<T>`, `AtomicLongFieldUpdater<T>`, `AtomicReferenceFieldUpdater<T,V>`)

| Method | Usage |
|---|---|
| `newUpdater(Class<T>, String fieldName)` | Static factory — creates the updater for a named `volatile` field of class `T`. `AtomicReferenceFieldUpdater` also needs the field's `Class<V>` |
| All methods above, object-qualified | Every method takes a leading `T obj` parameter instead of operating on "itself," e.g. `compareAndSet(T obj, expected, new)`, `incrementAndGet(T obj)` |

## `AtomicStampedReference<V>`

| Method | Usage |
|---|---|
| `getReference()` | Returns just the reference, ignoring stamp |
| `getStamp()` | Returns just the current stamp |
| `get(int[] stampHolder)` | Returns reference AND writes current stamp into `stampHolder[0]` — the idiomatic way to read both atomically |
| `compareAndSet(expectedRef, newRef, expectedStamp, newStamp)` | CAS on **both** reference and stamp together — this is the ABA fix |
| `attemptStamp(expectedRef, newStamp)` | Updates just the stamp if reference still matches, without changing the reference |
| `set(newRef, newStamp)` | Unconditional write of both |

## `AtomicMarkableReference<V>`

| Method | Usage |
|---|---|
| `getReference()` | Returns just the reference |
| `isMarked()` | Returns just the boolean mark |
| `get(boolean[] markHolder)` | Returns reference AND writes current mark into `markHolder[0]` |
| `compareAndSet(expectedRef, newRef, expectedMark, newMark)` | CAS on both reference and mark together — common for "logically deleted" flags in lock-free lists |
| `attemptMark(expectedRef, newMark)` | Updates just the mark if reference still matches |
| `set(newRef, newMark)` | Unconditional write of both |

## `LongAdder` / `DoubleAdder`

| Method | Usage |
|---|---|
| `add(x)` | Adds `x` to the striped total — the main write path, cheap under contention |
| `increment()` / `decrement()` | `LongAdder` only — shorthand for `add(1)`/`add(-1)` |
| `sum()` | Combines all internal stripes into a single total (relatively expensive — do sparingly, e.g. for reporting/logging, not on a hot path) |
| `reset()` | Resets all stripes to zero (not atomic as a whole — a concurrent write can be lost) |
| `sumThenReset()` | Atomically-ish reads the sum and resets in one call — safer than separate `sum()`+`reset()` |
| `longValue()` / `doubleValue()` (etc.) | Equivalent to `sum()`, exposed via the `Number` supertype |

## `LongAccumulator` / `DoubleAccumulator`

| Method | Usage |
|---|---|
| `accumulate(x)` | Combines `x` into the striped state using the constructor-supplied `LongBinaryOperator`/`DoubleBinaryOperator` (e.g., `Math::max`) |
| `get()` | Combines all stripes and returns the current accumulated result |
| `reset()` | Resets all stripes to the identity value given at construction |
| `getThenReset()` | Atomically-ish reads then resets |

## Quick usage notes worth remembering

- **`weakCompareAndSet*`** variants exist purely for performance on architectures using LL/SC (ARM) where a "weak" CAS avoids the retry-on-spurious-failure cost being pushed onto the JVM — meaningful only inside your own retry loops, not a general-purpose replacement for `compareAndSet`.
- **`compareAndExchange`** vs **`compareAndSet`**: prefer `compareAndExchange` when writing manual CAS-retry loops yourself, since it saves a redundant `get()` call on failure (you already have the witness value to retry with).
- **`AtomicBoolean`** deliberately has no `getAndUpdate`/`accumulateAndGet`/arithmetic methods — booleans have no natural "add" or general unary-function use case beyond toggling, which you'd do via plain `compareAndSet` in a loop.
- **`lazySet`** is a niche optimization — used almost exclusively in producer-side code of single-producer queues (e.g., inside `ConcurrentLinkedQueue`'s internals) where you know no other thread will race on the very next instruction.

## ABA Problem
- CAS asks "is the value still A?" — not "has the value been untouched?". Those differ.
- Thread 1 reads A. Threads 2 and 3 change it to B and then back to A. Thread 1's CAS now succeeds, because the value is A, even though the world moved on in between.
- For a counter this is harmless — a long that went 5 → 6 → 5 really is 5.
- It is dangerous where the value is a reference and identity matters. In a lock-free stack, a node popped, recycled and pushed back may be the same object at the same address while the list it belongs to has completely changed. The CAS succeeds and corrupts the structure.
- Solution: `AtomicStampedReference` pairs the reference with a counter that increments on every change, so A-with-stamp-7 and A-with-stamp-9 are distinguishable.

## What Atomics dont give:
- They do not compose. Two atomic operations in sequence are not atomic. The only escape is to make the two things one thing. 
- They do not cover multiple objects. If an operation must update a wallet and an order and append a ledger entry as one unit, no CAS covers that. You need a lock, or a transaction, or a redesign that makes one object the single point of truth. 
- They do not help with long operations. An optimistic retry loop assumes retrying is cheap. If the work between read and CAS is expensive, repeating it under contention is worse than waiting.

# Miscellaneous 
## ABA problem in reference variable

### The stack setup

```java
class Node<T> {
    final T value;
    Node<T> next;
    Node(T value, Node<T> next) { this.value = value; this.next = next; }
}

class LockFreeStack<T> {
    private final AtomicReference<Node<T>> top = new AtomicReference<>();

    public void push(T value) {
        Node<T> newHead;
        Node<T> oldHead;
        do {
            oldHead = top.get();
            newHead = new Node<>(value, oldHead);
        } while (!top.compareAndSet(oldHead, newHead));
    }

    public T pop() {
        Node<T> oldHead;
        Node<T> newHead;
        do {
            oldHead = top.get();
            if (oldHead == null) return null;
            newHead = oldHead.next;
        } while (!top.compareAndSet(oldHead, newHead));
        return oldHead.value;
    }
}
```

`compareAndSet(oldHead, newHead)` only checks **object identity/reference equality** — "is `top` still pointing at this exact `Node` object?" It has no idea whether that node's `.next` pointer still means what it meant a moment ago.

### The race

Stack contents: `top → A → B → C`

**Thread 1** calls `pop()`:
1. Reads `oldHead = A`
2. Reads `newHead = A.next = B`
3. **Gets paused right here** (OS preempts it, GC pause, whatever) — before it executes the CAS.

**Meanwhile, Threads 2 and 3 run to completion:**
4. Thread 2 pops `A` → stack is now `top → B → C`. Node `A` is now unreferenced by the stack.
5. Thread 2 pops `B` → stack is now `top → C`.
6. Thread 3 pushes some new value → creates a **new node**, but — critically — say it **reuses/recycles the same `Node` object `A`** (e.g., from an object pool, or the allocator just happens to hand back a Node object that, for our purposes, `equals`/`==` the old `A` — this is the part that requires special conditions in Java, more below). Now: `top → A → C` (a *reused* `A`, with `A.next` now pointing to `C`, not `B`).

**Thread 1 resumes:**
7. Executes `top.compareAndSet(A, B)`.
8. The CAS checks: "Is `top` currently `== A`?" — **Yes!** Same object reference. CAS succeeds.
9. `top` is now set to `B` — but `B` was already popped and possibly already garbage/reused elsewhere. The stack is now silently corrupted: `top → B → ???`, and `C` has been orphaned/lost, while `B` might not even be a valid node anymore.

Thread 1 had no way to detect that between its read and its CAS, the world changed from `A→B→C` to `A→C` (or worse) and back to an `A` that only *looks* like the one it saw before.

### Why this is "A-B-A" specifically

The value at the memory location went **A → (something else) → A**. Thread 1's CAS only compares the snapshot value against the current value — it can't tell "this is the same A as before, nothing happened" from "this is a *different* A that got recycled back into the same slot after a bunch of other activity happened in between." Same address/reference, completely different world.

### Does this actually happen in plain Java?

This is the subtle bit — worth being precise about, since Java has GC:

- In **C/C++** (or manual memory management generally), this is very real: `free(A)` followed by `malloc()` can literally hand back the same memory address for a new, unrelated object. Classic ABA setting.
- In **Java**, the GC guarantees that as long as a reference to an old `Node` exists anywhere (including possibly in a thread's local stale read, or in an object pool), that exact object cannot be collected and its identity can't be "stolen" by an unrelated allocation. So the *literal* "GC reused the address for something else" scenario doesn't happen the same way.
- **But** ABA still bites in Java whenever you have **object pooling/recycling by design** — e.g. a stack of `Node` objects that get pushed, popped, and explicitly returned to a **free list / object pool** for reuse (common in high-performance code to avoid allocation/GC pressure). If node `A` gets popped, returned to the pool, and then reused for a subsequent `push`, you get the exact same-object-different-meaning problem — not because the JVM reused an address, but because *your program logic* reused the same `Node` instance.

### The fix: `AtomicStampedReference` / `AtomicMarkableReference`

```java
AtomicStampedReference<Node<T>> top = new AtomicStampedReference<>(null, 0);
```

Every CAS also compares a **version stamp** (an incrementing int) alongside the reference:

```java
int[] stampHolder = new int[1];
Node<T> oldHead = top.get(stampHolder);
int oldStamp = stampHolder[0];
...
top.compareAndSet(oldHead, newHead, oldStamp, oldStamp + 1);
```

Now even if the reference comes back to being `A` again, the stamp has moved on (it was incremented on every successful mutation in between), so the CAS correctly fails — Thread 1 detects staleness and retries with a fresh read instead of corrupting the stack.

Want me to sketch the full stamped version of `push`/`pop` to make this concrete?

