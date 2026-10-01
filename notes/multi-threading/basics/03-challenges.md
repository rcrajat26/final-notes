All five are the same underlying disease — an operation that *looks* like one step in source code is actually several steps at runtime, and another thread gets scheduled in the gap. The differences are just where the gap sits and what invariant it violates.

## 1. read-modify-write → lost update

`balance += amount` compiles to (roughly) `GETFIELD balance` → add → `PUTFIELD balance`. Three steps, not one.

```
T1: read balance = 100
                        T2: read balance = 100
T1: compute 100+50 = 150
                        T2: compute 100+30 = 130
T1: write balance = 150
                        T2: write balance = 130   <- clobbers T1's update
```

Final balance is 130. The +50 deposit vanished. Neither thread did anything wrong in isolation — the bug is that "read" and "write" aren't paired atomically.

## 2. check-then-act → overdraft

`if (balance >= amt) debit(amt)` — the check and the act are separate operations with a window between them (classic TOCTOU).

```
T1: check 100 >= 80 → true
                        T2: check 100 >= 80 → true
T1: debit 80 → balance = 20
                        T2: debit 80 → balance = -60
```

Both threads saw a "safe" snapshot, both proceeded, and the invariant `balance >= 0` breaks because the check's truth expired before the act happened.

## 3. put-if-absent → duplicate order

`if (!orders.containsKey(id)) orders.put(id, o)` — same TOCTOU shape as #2, just on a map instead of a number.

```
T1: containsKey(id) → false
                        T2: containsKey(id) → false
T1: put(id, orderA)
                        T2: put(id, orderB)   <- overwrites orderA silently
```

Worse than an exception: nothing crashes, one order just disappears, and both callers believe *they* were the one who successfully created it — because each one's local check said "not present" right before it acted.

## 4. iterate-then-act → acts on a stale list

Scanning resting orders and then executing matches against that scan means the list you're iterating and the list at execution time can diverge.

```
T1: scans resting orders → snapshot [O1, O2, O3]
                        T2: cancels O2
T1: executes match against O2   <- O2 no longer should be tradable
```

This isn't lost-update or overdraft, it's a **stale view**: the data you're acting on was true at scan time but the world moved on before you finished acting on it. (If the iteration is over a plain `HashMap`/`ArrayList` and T2 structurally mutates it mid-scan, you also get `ConcurrentModificationException` — a "lucky" fast-failure version of the same bug. The dangerous version is when nothing throws and you silently trade against a cancelled order.)

## 5. multi-variable invariant → fields disagree

Moving `available -= amt; reserved += amt` is two separate writes. Even if each individual field is `volatile` or an `AtomicLong`, that only makes *each write* atomic — it says nothing about the *pair*.

```
T1: available -= 100   (available: 500→400, reserved still 0)
                        T3: reads available=400, reserved=0 → total looks like 400, not 500
T1: reserved += 100    (reserved: 0→100)
```

For that instant, T3 observed a world where $100 evaporated — `available + reserved` invariant is violated. This is the same lost-update family but across *two* memory locations instead of one, so no amount of per-field atomicity fixes it. You need the pair itself treated as a single unit.

## The common root cause and the fix family

All five are "composite operation, not actually indivisible." The fix is always some form of **making the whole thing one atomic unit**, and you've basically already built two of the mechanisms in your wallet app:

- **Mutual exclusion** — `synchronized`, `ReentrantLock`, or `SELECT ... FOR UPDATE` at the DB level: serialize the whole read-check-write sequence so no other thread can see the intermediate state.
- **Optimistic concurrency** — `@Version` / CAS loop (`AtomicReference.compareAndSet`, `AtomicLong.compareAndSet`): let both threads race through the read, but make the *write* fail and retry if the version/value moved underneath it.
- **Atomic composite structures** — `ConcurrentHashMap.putIfAbsent` (fixes #3 directly by fusing check+act into one atomic call), `ConcurrentHashMap.compute`/`merge` (fixes #1), copy-on-write / immutable snapshots for #4 (freeze a consistent view, and validate-before-commit rather than act-on-stale).
- For #5 specifically: the two field writes need to be under the *same* lock/transaction, or collapsed into one state object (`Ledger(available, reserved)`) that's swapped atomically via CAS — no amount of making `available` and `reserved` individually thread-safe helps, because the invariant lives *between* them, not inside either one.

Worth internalizing: `volatile` and individual atomics solve visibility and single-variable atomicity. None of these five bugs are visibility bugs — they're all "correct value, wrong moment," which only mutual exclusion or optimistic-retry-with-validation can fix.

---

Let's go through each fix mechanism properly — mechanics, code, and where it actually holds up vs. where it quietly doesn't.

## 1. Mutual exclusion — serialize the composite operation

The idea: make the read-check-write sequence into a **critical section** so no other thread can observe or act during the gap. This doesn't make the operation atomic in the CPU sense — it makes it *appear* atomic by forbidding interleaving.

**`synchronized` — fixes read-modify-write (#1) and check-then-act (#2)**

```java
public class Wallet {
    private long balanceCents;

    public synchronized void deposit(long amount) {
        balanceCents += amount;   // read-modify-write, now guarded
    }

    public synchronized boolean withdraw(long amount) {
        if (balanceCents < amount) return false;   // check
        balanceCents -= amount;                     // act
        return true;
    }
}
```

Every thread has to acquire the monitor on `this` before entering either method. T2 can't even *read* `balanceCents` mid-update from T1 because entering `synchronized` also establishes a happens-before edge — you get visibility for free, not just exclusion. The cost: every deposit/withdraw across *all* wallets sharing this lock serializes, even ones with no logical conflict. For a single wallet object that's usually fine (one customer isn't depositing and withdrawing 10,000 times a second), but if you did this at a service level with one shared lock for all wallets, you'd be serializing unrelated customers for no reason.

**`ReentrantLock` — same guarantee, more control**

Same semantics as `synchronized`, but you get `tryLock(timeout)`, `lockInterruptibly()`, and separate condition queues. Useful when you don't want to block forever waiting for a contested wallet — e.g. fail fast and tell the mobile client "try again" rather than hang the request thread:

```java
private final ReentrantLock lock = new ReentrantLock();

public boolean withdraw(long amount) {
    if (!lock.tryLock(200, TimeUnit.MILLISECONDS)) {
        throw new WalletBusyException();
    }
    try {
        if (balanceCents < amount) return false;
        balanceCents -= amount;
        return true;
    } finally {
        lock.unlock();
    }
}
```

**`SELECT ... FOR UPDATE` — the DB-level version, fixes #1/#2/#5 across processes**

`synchronized` only protects threads inside *one JVM*. The moment you have two Fargate tasks running the same wallet-service, in-memory locks are useless — T1 and T2 aren't even in the same address space. You need the lock to live where the shared state lives: the row.

```sql
BEGIN;
SELECT available, reserved FROM wallet WHERE id = :id FOR UPDATE;
-- row is now locked; any other transaction's SELECT ... FOR UPDATE
-- on this row blocks until we COMMIT/ROLLBACK
UPDATE wallet SET available = available - :amt, reserved = reserved + :amt WHERE id = :id;
COMMIT;
```

This is exactly what fixes the `available`/`reserved` invariant problem (#5) correctly — both fields move inside one transaction holding the row lock, so no reader using `FOR UPDATE` (or under `SERIALIZABLE`/`REPEATABLE READ` depending on engine) can observe the half-moved state. This is presumably the pessimistic side of the locking pair you already built in your Spring Boot wallet app — the JPA equivalent is `@Lock(LockModeType.PESSIMISTIC_WRITE)` on the repository method, which Hibernate translates into that same `FOR UPDATE` clause at flush/query time.

Trade-off: lock is held for the *duration of the transaction*, including any network round-trip or business logic in between. Long transactions holding row locks under concurrent load is how you get connection-pool exhaustion and lock-wait timeouts — this is the classic argument for keeping the transaction as short as possible, or moving to optimistic locking when contention is low.

## 2. Optimistic concurrency — race through, fail only on write

Instead of blocking readers, let everyone read freely and only detect conflict at the moment of write. Good when conflicts are *rare* — most of the time nobody's actually contending, so paying a lock's serialization cost on every access is wasted.

**`@Version` — fixes #1 at the DB/entity level**

```java
@Entity
class Wallet {
    @Id Long id;
    long balanceCents;
    @Version int version;
}
```

Hibernate turns every `UPDATE` into a version-checked conditional write:

```sql
UPDATE wallet SET balance_cents = ?, version = version + 1
WHERE id = ? AND version = ?   -- the version you loaded, not the current one
```

If T2 already bumped `version` between your load and your flush, this `UPDATE` matches zero rows, Hibernate notices `rowsAffected == 0`, and throws `OptimisticLockException`. Nothing was corrupted — T1's write simply never happened; the caller catches it and retries the whole read-modify-write with a fresh load. This is the optimistic half of the pair in your wallet app — same read-modify-write bug as #1, opposite strategy from `FOR UPDATE`: don't prevent the race, detect it after the fact and retry.

**CAS loop — the hand-rolled version, no DB needed**

`AtomicLong` doesn't give you `+=` as one op either, but it gives you `compareAndSet`, and the idiom builds atomicity out of a *retry loop* instead of a lock:

```java
private final AtomicLong balanceCents = new AtomicLong(10_000);

public void deposit(long amount) {
    long prev, next;
    do {
        prev = balanceCents.get();
        next = prev + amount;
    } while (!balanceCents.compareAndSet(prev, next));  // retry if someone else moved it
}
```

`compareAndSet(prev, next)` is a single CPU instruction (`cmpxchg` on x86) — it atomically checks "is the current value still `prev`?" and if so swaps to `next`, all in one indivisible hardware step. If T2 wrote in between, `compareAndSet` returns `false`, T1's loop just re-reads and tries again. No thread ever blocks; under low contention this is dramatically cheaper than `synchronized` because there's no OS-level mutex, no context switch — just a spin-and-retry that usually succeeds on the first try. Under *high* contention it degrades badly (everyone keeps invalidating everyone else's CAS), which is exactly why `LongAdder` exists for hot counters — but for a wallet balance under normal load, plain CAS is the right tool.

Note this fixes #1 only. It does **not** fix #2 (check-then-act) by itself — `compareAndSet` only guards the *write*, not a check that happened earlier against a value that's now stale. For check-then-act you need the check baked into the CAS condition itself, which is really just the CAS loop again with a guard:

```java
public boolean withdraw(long amount) {
    long prev, next;
    do {
        prev = balanceCents.get();
        if (prev < amount) return false;      // check happens fresh, every retry
        next = prev - amount;
    } while (!balanceCents.compareAndSet(prev, next));
    return true;
}
```

The key difference from the buggy version: the check (`prev < amount`) is re-evaluated against a freshly-read value on *every* iteration, not evaluated once against a value that might be stale by the time you act.

## 3. Atomic composite structures — fuse check+act into one call

Rather than locking around two calls (`containsKey` then `put`), use a data structure that offers the *combined* operation as a single atomic method, so there's no gap to race in.

**`ConcurrentHashMap.putIfAbsent` — fixes #3 directly**

```java
ConcurrentMap<String, Order> orders = new ConcurrentHashMap<>();

Order existing = orders.putIfAbsent(id, newOrder);
if (existing != null) {
    // someone already created this order id — don't overwrite, don't duplicate
    throw new DuplicateOrderException(id);
}
```

Internally, `ConcurrentHashMap` takes a lock on just the *bucket* for that key (or uses CAS on the bin head, depending on JDK version) for the duration of the check-and-insert — but that locking is implementation detail hidden behind the method. From your side, `containsKey`-then-`put` became one call, so there's no window for another thread to interleave. This is the whole point of "atomic composite structures": push the mutual-exclusion problem down into a library that's already solved it correctly, instead of hand-rolling it above the data structure.

**`compute` / `merge` — fixes #1 for map-shaped values**

If your wallet balances live as map entries (e.g. `Map<AccountId, Long> balances` for multiple sub-ledgers) rather than one field:

```java
balances.merge(accountId, depositAmount, Long::sum);
```

`merge` reads, applies the function, and writes back all under one atomic operation per key — same shape as the CAS-loop deposit above, but the retry logic is internal to the map instead of something you write yourself.

**Copy-on-write / immutable snapshot — fixes #4 (iterate-then-act)**

The stale-list bug happens because you're iterating a *live, mutable* structure while deciding what to act on. The fix is to take an immutable snapshot to iterate, and **revalidate each item at the point of acting**, rather than trusting the snapshot to still be true:

```java
List<Order> restingSnapshot = List.copyOf(restingOrders.values()); // frozen view

for (Order o : restingSnapshot) {
    // don't act on the snapshot's belief — re-check the live source of truth
    Order current = restingOrders.get(o.id());
    if (current == null || current.status() != OPEN) {
        continue;  // it was cancelled/filled after our snapshot — skip
    }
    tryExecuteMatch(current);
}
```

This doesn't prevent T2 from cancelling `O2` mid-scan — it can't, and shouldn't try to, since blocking every cancel for the duration of a full order-book scan would be terrible for latency. Instead it accepts that the snapshot goes stale and **re-validates against fresh state right before acting**, which turns "acts on stale data" into "acts on a stale *candidate list*, but validates before commit" — the actual invariant (never execute a cancelled order) is preserved even though the iteration view isn't perfectly live. `CopyOnWriteArrayList` gives you this snapshot-iteration behavior automatically (iterators see a fixed snapshot from when the iterator was created), but it's a poor fit for a resting-order book specifically because every cancel/insert copies the entire underlying array — fine for rarely-mutated listener lists, bad for a structure mutated on every order.

## 4. Composite state object — fixing invariants that span fields

This is the one that most people get wrong: they make `available` an `AtomicLong` and `reserved` an `AtomicLong` separately, and believe they've fixed #5. They haven't — each field is internally atomic, but the *pair* isn't, because there's no operation that moves both under one CAS.

The fix: stop having two fields. Have one immutable object representing the pair, and CAS the *whole object*:

```java
record LedgerState(long available, long reserved) {}

private final AtomicReference<LedgerState> ledger =
    new AtomicReference<>(new LedgerState(500_00, 0));

public void reserve(long amount) {
    LedgerState prev, next;
    do {
        prev = ledger.get();
        if (prev.available() < amount) throw new InsufficientFundsException();
        next = new LedgerState(prev.available() - amount, prev.reserved() + amount);
    } while (!ledger.compareAndSet(prev, next));
}
```

Any reader calling `ledger.get()` sees either the state *before* the move or the state *after* it — `available + reserved` is always consistent from any observer's point of view, because there is no moment where one field is updated and the other isn't; the swap of the whole `LedgerState` reference is the atomic unit, not the individual `long`s inside it. This is the same principle as `@Version` optimistic locking, just applied to in-memory state instead of a DB row — the "version" here is implicit in object identity (`prev` is only valid if `ledger.get()` still returns that exact reference).

The DB equivalent of this is just #1's `FOR UPDATE`/`@Version` pattern applied to a row that happens to have two columns — the transaction (or the version check) covers both column writes as one unit, which is why `SELECT ... FOR UPDATE` was the correct fix for #5 in the mutual-exclusion section above. The `AtomicReference<record>` pattern is the same idea for wallet state you're keeping in memory rather than persisting on every mutation.

## Which fix for which bug — summary

| Bug | Lock-based fix | Lock-free fix |
|---|---|---|
| #1 read-modify-write | `synchronized`, `FOR UPDATE` | CAS loop, `@Version`, `merge` |
| #2 check-then-act | `synchronized` with check inside | CAS loop with check re-evaluated per retry |
| #3 put-if-absent | lock around containsKey+put | `ConcurrentHashMap.putIfAbsent` |
| #4 iterate-then-act | lock the whole scan+act (kills concurrency) | snapshot + revalidate-before-commit |
| #5 multi-variable invariant | `FOR UPDATE` covering both columns | `AtomicReference<CompositeRecord>` CAS |

The general lesson underneath all of it: you can't fix "the gap between steps" by making each step individually safe. You either close the gap (lock it shut) or you make the *whole* multi-step thing swappable as one unit (CAS the composite, version the row) so a conflict is detected and retried rather than silently corrupting state.

---

### The three properties

**Atomicity — does this happen all at once?**
Can another thread observe a half-finished operation, or slip in between my steps? Both bugs in this file are atomicity failures. Fixed by locks or by atomic classes.


**Visibility — will another thread ever see my write?**
I set `balance = 52000`. Is there any guarantee another thread reading `balance` a second later sees anything other than the old value? There is not, by default. The write may sit in a CPU store buffer, or the JIT may have kept the value in a register and never written it to memory at all. Fixed by `volatile`, locks, or `final`.

**Ordering — do my operations appear to happen in the order I wrote them?**
The compiler, the JIT and the CPU all freely reorder instructions, as long as the executing thread cannot tell. Another thread absolutely can tell. Fixed by `volatile`, locks, or `final`

More explanation on this:
```
// Thread A
data = 42;
ready = true;

// Thread B
while (!ready) { }
print(data);
```

You'd expect Thread B to print 42. But without proper synchronization:
- The compiler/CPU might reorder Thread A's writes, so ready becomes visible to Thread B before data does.
- Thread B might print 0 (or garbage) even though ready is true.


### What each tool actually gives you
| Mechanism | Atomicity | Visibility | Ordering |
|---|---|---|---|
| plain field | ✗ | ✗ | ✗ |
| `volatile` field | ✗ | ✓ | ✓ |
| `AtomicLong` etc. | ✓ *(one variable)* | ✓ | ✓ |
| `synchronized` / `Lock` | ✓ *(whole block)* | ✓ | ✓ |
| `final` field | — | ✓ *(after construction)* | ✓ |

---
### Volatile
- It's a keyword you put on a field, It's a visibility and ordering guarantee for that one variable.
  - Visibility: Every write to a volatile field goes straight to main memory (not just a CPU cache/register), and every read comes straight from main memory. So no thread is stuck looking at a stale cached copy.
  - Ordering: The compiler and CPU are forbidden from reordering reads/writes of volatile fields with respect to other volatile reads/writes. This means that if one thread writes to a volatile field and then another thread reads it, the second thread is guaranteed to see all the writes that happened before the first thread's write. Meaning in the previous example, ready=true implies that data=42 has already happened, so Thread B will see data=42 when it sees ready=true.

### AtomicLong (and friends: AtomicInteger, AtomicReference, etc.)
- A wrapper class that makes read-modify-write sequences atomic, using CAS — Compare-And-Swap — instead of locks.
- CAS is a single hardware instruction that says "if the value is still X, change it to Y; otherwise, do nothing." No thread can observe a half-finished state.. This allows you to implement atomic operations without locking, but you have to handle the case where the value changed between your read and your write (usually by retrying).

```java
AtomicLong counter = new AtomicLong(0);
counter.incrementAndGet();
```

```java
long prev, next;
do {
    prev = get();          // read current value
    next = prev + 1;
} while (!compareAndSwap(prev, next));  // retry if someone else changed it
```

---
# Miscellaneous 
##  A thread that never sees the write

The wallet system runs a background sweeper that expires stale limit orders. It needs to stop cleanly at shutdown, so it spins on a flag:

```java
class OrderSweeper implements Runnable {
    private boolean running = true;

    public void run() {
        while (running) {
            expireStaleOrders();
        }
    }

    public void shutdown() {
        running = false;     // called from another thread
    }
}
```

This is code you have written. It is obviously correct: one thread sets a boolean, the other reads it.

### What actually happens

Three variants, one JVM, one run. A thread spins on a flag; after 500 ms the main thread sets it to `false`; we then wait up to three seconds for the spinner to notice.

```
1. plain field                             STILL SPINNING after 3s  <-- never saw the write
2. volatile field                          stopped
3. plain field + synchronized block in loop stopped
```

**Variant 1 never stops.** Not "stopped late". It would spin until the process was killed. The write happened. The spinning thread read that exact field, several billion times, and saw the old value every single time.

### Why
**The JIT compiler hoists the read out of the loop**

After a few thousand iterations, compiler decides the loop is hot and compiles it to machine code. While doing so, it asks a reasonable question: *does anything inside this loop modify `running`?* No. So the read is loop-invariant, and re-reading it from memory on every iteration would be wasted work. It transforms your loop into something equivalent to:

```java
boolean copy = running;        // read ONCE
while (copy) {
    expireStaleOrders();       // now an infinite loop, permanently
}
```

This is a completely legal optimisation. The rule the compiler must obey is *as-if-serial*:the optimised code must produce the same result **for the thread executing it**. Within this thread, nothing writes `running`, so the transformation is sound. The compiler has no obligation to consider other threads unless you tell it to — and the way you tell it is`volatile`.

**Store buffers.** A CPU core does not write directly to memory; writes go into a store buffer and drain asynchronously. This delays visibility, but only by nanoseconds. It causes reordering rather than a permanent failure to observe. ![refer below](#store-buffers)

> The honest summary: **the Java Memory Model gives you no visibility guarantee at all for
> plain fields, and the JIT compiler is what actually cashes in on that freedom.** The bug
> is a compiler optimisation, not a hardware artefact.

### 1.4 Why variant 3 is worth staring at
```java
public void run() {
    while (running) {
        synchronized (someLock) {
            // does nothing relevant to `running`
        }
        expireStaleOrders();
    }
}
```
Variant 3 has the same plain, non-volatile field. The only change is a `synchronized` block inside the loop that does nothing relevant. And it stops.

A monitor enter and exit is a memory barrier. The compiler is not permitted to hoist a field read across one, so it must re-read `running` every iteration.

In variant 3, the JIT sees a synchronized block sitting inside the loop. It doesn't know what that lock might guard elsewhere, so as a conservative safety measure it refuses to hoist any memory read across that monitor boundary — not just reads related to the lock, but any field read nearby, including running. 

This is the mechanism behind a class of bug that wastes enormous amounts of engineering time:

> **[TRAP]** *"I added a `System.out.println` to debug it and the problem went away."*
> `PrintStream.write` is `synchronized`. Adding it to a loop inserts a memory barrier,
> which forces the re-read, which makes the bug disappear. Attaching a debugger does the
> same thing more comprehensively.
>
> **The bug is not gone. It is hidden, and it will return when the logging is removed.**
> Any time a concurrency bug vanishes on being observed, assume a visibility problem.

### Store buffers
Modern CPUs don't write directly to the cache/memory hierarchy the instant a store instruction executes. Instead:
- CPU executes x = 1.
- Instead of blocking and waiting for that write to travel all the way to L1/L2/L3 cache (which, relatively speaking, takes a while), the CPU shoves the write into a small, very fast, per-core buffer called the store buffer.
- The CPU immediately moves on to the next instruction — it doesn't wait.
- Meanwhile, in the background, the store buffer feeds that pending write out to the actual cache/memory system, whenever there's a free cycle to do so.

**How this causes reordering**
```
data = 42;      // store 1 → goes into store buffer
ready = true;   // store 2 → goes into store buffer
```

Both writes land in the store buffer. But the store buffer doesn't necessarily drain them to memory in strict FIFO order, and even if it did, the timing of when each becomes visible to other cores can differ.

So it's entirely possible that:
- ready = true drains and becomes visible to other cores quickly (its cache line was free). 
- data = 42 is still sitting in the store buffer, not yet visible (its cache line needed to be fetched from another core first).

In second thread:
```
while (!ready) { }   // sees ready == true (this one drained already)
print(data);          // but data write hasn't drained yet — sees 0, not 42!
```

**Why this connects back to volatile/locks**
A memory barrier forces the store buffer to drain — flush its pending writes out to cache — before the next instruction executes, or before a lock is released, etc. That's literally what closes this gap: it converts "eventually visible, in some order" into "visible now, in this specific order," at the cost of that flush taking real time (which is part of why synchronized/atomic operations are slower than plain reads/writes).


