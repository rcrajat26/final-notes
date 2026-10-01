# Mutual exclusions: synchronization and locks 
- Mutual exclusion is a property of concurrent programming that ensures that only one thread can access a shared resource at a time. This is important to prevent race conditions, where multiple threads try to modify the same resource simultaneously, leading to unpredictable behavior.
- 
## The three properties

**1. Atomicity**
Definition: An operation is atomic if it appears to happen as a single, indivisible step — other threads never see it "half-done."

```
counter++;
```

This looks like one operation, but compiles to three steps:
- Read counter into a register 
- Add 1 
- Write it back

If two threads interleave these steps, one increment can be lost:
```
Thread A: reads counter = 5
Thread B: reads counter = 5
Thread A: writes counter = 6
Thread B: writes counter = 6   // A's increment is lost!
```
How to fix it:
- Locks/mutexes: synchronized block, pthread_mutex, etc. — force one thread at a time.
- Atomic classes: AtomicInteger, std::atomic<int> — use CPU-level instructions like Compare-And-Swap (CAS) to make read-modify-write a single hardware-guaranteed step.
- Immutability: if data never changes, there's nothing to race over.

**2. Visibility**
Definition: Visibility is about whether a write made by one thread is actually seen by another thread — and when.

Why this is a real problem: CPUs and compilers cache values in registers or per-core caches for performance. A write from Thread A might sit in A's CPU cache and never get flushed to main memory (or Thread B might be reading from its own stale cache line).

```java
boolean flag = false;

// Thread A
flag = true;

// Thread B
while (!flag) { /* spin */ }   // may loop forever!
```

Without a visibility guarantee, Thread B might never observe flag becoming true, because the compiler cached it in a register or reordered/optimized the loop assuming nothing else changes it.

How to fix it:
- volatile (Java) / equivalent memory-mapped guarantees: forces reads/writes to go through main memory, not a cached copy.
- Locks: acquiring/releasing a lock also flushes changes.
- Atomic variables: typically come with visibility guarantees built in.
- Memory barriers/fences: explicit low-level instructions that force synchronization between cache and memory.

**3. Ordering**
Definition: Ordering is about whether operations execute (and are observed by other threads) in the sequence you wrote them in.

Why this is a real problem: Both compilers and CPUs reorder instructions for performance, as long as it doesn't change the outcome for a single thread running alone. But in a multithreaded context, another thread can observe the reordering.

```java
// Thread A
data = 42;        // (1)
ready = true;      // (2)

// Thread B
if (ready) {        // (3)
    print(data);     // (4)
}
```

You'd expect: if B sees ready == true, then data must already be 42. But if the compiler/CPU reorders (1) and (2), B could see ready = true while data is still 0 — a bug that's incredibly hard to reproduce.

How to fix it:
- Memory barriers / fences: prevent reordering across a certain point.
- volatile (Java): guarantees not just visibility but also a happens-before ordering — writes before a volatile write are visible/ordered before reads after the corresponding volatile read.
- Locks: also establish ordering, not just mutual exclusion.
- Language memory models (Java Memory Model, C++11 memory model): formally define what reorderings are legal and what synchronization primitives restore ordering.

A lock provides **mutual exclusion**: at most one thread may hold it at a time. Everything else follows from that.

| Problem | How a lock solves it |
|---|---|
| **Atomicity** | No other thread can be inside the same lock, so nobody observes your intermediate states |
| **Visibility** | Releasing a lock flushes your writes; acquiring it makes another holder's writes visible |
| **Ordering** | Lock acquire and release are memory barriers; the compiler and CPU may not move accesses across them |

> Notice the words that same lock. Two threads synchronising on different objects have no happens-before relationship at all. They are not mutually excluded, and they see nothing of each other's writes. This is why "what object am I locking on" is the question that decides whether your code works.

---

## `synchronized`
- Every Java object has a monitor associated with it — a hidden lock, one per object. synchronized acquires it on entry and releases it on exit, including when an exception is thrown.

```java
class Wallet {
    // (1) instance method -> locks on `this`
    synchronized void credit(long p) { ... }

    // (2) static method -> locks on Wallet.class, NOT on any instance
    static synchronized void resetGlobalCounter() { ... }

    // (3) block -> locks on whatever you name
    void credit(long p) {
        synchronized (someObject) { ... }
    }
}
```

### The previous bugs fixed
```java
class Wallet {
    private long balancePaise;
    synchronized void credit(long p) { balancePaise += p; }
    synchronized long balance()      { return balancePaise; }
}
```

The whole read-compute-write is now inside one lock, so no thread can read between another thread's read and write.

## What to lock on
- The object you lock on is a design decision with real consequences. Three rules.

### 1. Never lock on a publicly reachable object
```java
synchronized void credit(long p) { ... }     // locks on `this`, which is public
```

- `this` is reachable by anyone holding a reference to your wallet. 
- Any code anywhere can write `synchronized (wallet) { ... }` and hold your lock for as long as it likes. 
- You have made your internal locking protocol part of your public API without documenting it, and you cannot reason about who holds it.

```java
class Wallet {
    private final Object lock = new Object();     // private AND final

    void credit(long p) {
        synchronized (lock) { balancePaise += p; }
    }
}
```

- `final` matters as much as `private`. If the lock reference can be reassigned, two threads can synchronise on two different objects and neither excludes the other.

### 2. Never lock on a String or a boxed primitive
```java
synchronized (userId) { ... }                 // userId is a String. Do not do this.
synchronized (accountNumber) { ... }          // an Integer. Also do not.
```

- Strings and boxed primitives are cached and reused by the JVM. Two different pieces of code might synchronise on the same String or Integer without knowing it, leading to accidental contention or deadlocks.
- To use lock per user:
```java
private final ConcurrentMap<String, Object> walletLocks = new ConcurrentHashMap<>();

Object lockFor(String userId) {
    return walletLocks.computeIfAbsent(userId, k -> new Object());
} 
```

### 3. One lock per invariant, not one lock per object
If a class protects two unrelated pieces of state, one lock makes them contend for no reason. If it protects one invariant spanning several fields, they must share a lock. The grouping follows the invariants, not the field declarations.

An invariant is a relationship between two or more pieces of state that must hold true at every point where another thread can observe that state. It's not about a single variable — it's a constraint across variables (or across objects) that must never be caught in a half-updated condition.

Examples of invariants:
- lower <= upper (a range)
- list.size() == count (a cache/count kept in sync with a collection)
- balance == sum(transactions)
- x*x + y*y == radiusSquared (derived/cached value matching source data)
- "money removed from account A equals money added to account B" (an invariant spanning two separate objects)

## Reentrancy of locks
- Java's locks are reentrant: a thread already holding a lock can acquire it again.
```java
class Wallet {
    public synchronized void outer() {
        inner(); // same thread trying to re-enter a lock it already owns
    }

    public synchronized void inner() {
        System.out.println("inner");
    }
}
```

- If synchronized weren't reentrant, calling outer() would acquire the lock on `this`, then calling inner() from inside outer() would try to acquire the same lock again — and since the thread would be waiting on a lock it itself holds, it would block forever.
- The JVM tracks an owner and a hold count: entering increments, exiting decrements, and the lock is released only at zero.
- Reentrancy is what makes `synchronized` usable with inheritance and with any method that calls another method on the same object.

**How it works internally**
A reentrant lock keeps two pieces of state instead of just "locked/unlocked":
- Owner thread — which thread currently holds the lock
- Hold count — how many times that thread has acquired it

Conceptually:
```
acquire(lock):
    if lock.owner == currentThread:
        lock.holdCount++          // just increment, don't block
    elif lock.owner == null:
        lock.owner = currentThread
        lock.holdCount = 1
    else:
        block until available

release(lock):
    lock.holdCount--
    if lock.holdCount == 0:
        lock.owner = null         // fully released only now
```

- So the lock is only truly released when the hold count returns to zero — meaning every lock() needs a matching unlock() (or every synchronized block needs to exit) the same number of times it was entered.
- Each synchronized method exit decrements the counter; the monitor is only released when it hits zero. Hence, its follows implicit reentrancy

**ReentrantLock — explicit reentrancy**
```java
import java.util.concurrent.locks.ReentrantLock;

public class Counter {
    private final ReentrantLock lock = new ReentrantLock();
    private int count = 0;

    public void increment() {
        lock.lock();
        try {
            count++;
            logChange();
        } finally {
            lock.unlock();
        }
    }

    public void logChange() {
        lock.lock();
        try {
            System.out.println("count = " + count);
        } finally {
            lock.unlock();
        }
    }
}
```
Same idea, but explicit. You can even inspect the hold count:
```
lock.lock();
lock.lock();
System.out.println(lock.getHoldCount()); // 2
lock.unlock();
lock.unlock();
```

### The general rule
Make the critical section as short as possible, but no shorter than the invariant requires.
- Too wide: you serialise work that did not need serialising, and you import the failure modes of anything you call. 
- Too narrow: the invariant breaks, and you are back to file 02.
Practically: compute outside the lock, mutate inside it.

```java
// bad — expensive work under the lock
synchronized (lock) {
    var fee = feeEngine.calculate(order);     // expensive, touches nothing shared
    available -= fee;
}

// good
var fee = feeEngine.calculate(order);          // outside
synchronized (lock) { available -= fee; }      // inside, briefly
```

### synchronized
`synchronized` is the right default. Four things it cannot do:
- **It cannot give up** `synchronized` blocks forever. There is no "try for 100 ms, then do something else".
- **It cannot be interrupted** A thread blocked on `synchronized` ignores `interrupt()` completely. It cannot be cancelled, and it cannot participate in a graceful shutdown. 
- **It must be block-structured** Acquire and release are tied to a lexical block(code block within brackets{}). A hand-over-hand traversal — lock the next node, then unlock the previous — cannot be written with it. 
- **It offers no fairness and only one wait set** Threads acquire in whatever order the JVM chooses (there is no guarantee like the next thread waiting gets it or FIFO order), and a monitor has a single wait/notify queue, so you cannot separately signal "space became available" and "an item arrived".

### ReentrantLock
Same semantics, explicit API, and the four capabilities above
- synchronized releases on exception automatically. ReentrantLock does not. Forgetting finally means one exception leaks the lock permanently and every subsequent thread hangs.
- It can be interruptible, and it can time out. It can have multiple wait sets (Condition objects). It can be fair (FIFO) instead of whatever the JVM chooses.
```java
ReentrantLock lock = new ReentrantLock();

// Option A: try immediately, don't wait at all
if (lock.tryLock()) {
    try {
        System.out.println("Got it immediately");
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("Someone else has it — doing something else instead");
}

// Option B: try for a bounded time, then give up
try {
    if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
        try {
            System.out.println("Got it within 100ms");
        } finally {
            lock.unlock();
        }
    } else {
        System.out.println("Gave up after 100ms — doing fallback work");
    }
} catch (InterruptedException e) {
    // handle cancellation
}
```
- ReentrantLock gives you a way to block that can be woken up by an interrupt:
```java
ReentrantLock lock = new ReentrantLock();

public void doWork() {
    try {
        lock.lockInterruptibly(); // can be interrupted while waiting
        try {
            // critical section
        } finally {
            lock.unlock();
        }
    } catch (InterruptedException e) {
        System.out.println("Interrupted while waiting for lock — cancelling");
        Thread.currentThread().interrupt(); // restore interrupt status
    }
} 
```
- If another thread calls .interrupt() on the thread that's blocked in lock.lockInterruptibly(), it throws InterruptedException immediately instead of continuing to wait. This makes graceful shutdown and cancellation possible — you can tell a stuck thread "stop waiting, we're shutting down" — which is simply not possible with synchronized. (Note: Note: plain lock.lock() on a ReentrantLock is still non-interruptible)
- Below creates a fair lock, the true flag param makes it a fair lock, which means threads acquire it in FIFO order instead of whatever the JVM chooses. This is useful for preventing starvation in some scenarios, but it can hurt throughput because it reduces the JVM's ability to optimise lock acquisition:
```java
ReentrantLock fairLock = new ReentrantLock(true); 
```

### `Condition` — waiting for something to become true
- A lock lets you wait for access. A Condition lets you wait for a state.
```java
private final ReentrantLock lock = new ReentrantLock();
private final Condition fundsAvailable = lock.newCondition();

void withdrawBlocking(long amt) throws InterruptedException {
    lock.lock();
    try {
        while (available < amt) {        // while, never if
            fundsAvailable.await();      // releases the lock while waiting
        }
        available -= amt;
    } finally { lock.unlock(); }
}

void credit(long amt) {
    lock.lock();
    try {
        available += amt;
        fundsAvailable.signalAll();      // wake the waiters
    } finally { lock.unlock(); }
} 
```
- It does the same fundamental job — let a thread release a lock and sleep until some condition becomes true, then wake back up holding the lock again.
- Each call to `newCondition()` creates a separate wait queue tied to that lock. You can create as many as you need.
- `await()` releases the lock while waiting and reacquires it before returning. That is what stops it deadlocking, and it is the key difference from `Thread.sleep()`, which holds every lock it owns.
- What await() does internally, step by step:
  - Atomically releases the lock (fully — even multiple locks, it releases all holds and remembers the count). 
  - Parks the thread on this condition's specific wait queue. 
  - When woken (by signal()/signalAll(), a timeout, or an interrupt), it re-acquires the lock before returning — restoring the same hold count it released. 
  - Only then does control return to your code, right after the await() call.
- The while loop is mandatory:
  - Spurious wakeups — the JDK permits a thread to wake up from await() without anyone calling signal(). Rare, but legally possible. 
  - Stolen conditions — even after a legitimate signal(), by the time this thread re-acquires the lock, another thread might have gotten in first and already consumed whatever made the condition true.
- signal() vs signalAll()
  - signal() wakes exactly one thread waiting on that condition
  - signalAll() wakes every thread waiting on that condition.
- Interruption behavior 
  - By default, await() is interruptible — like lockInterruptibly(), if another thread calls .interrupt() on a thread parked in await(), it throws InterruptedException and the thread does not silently continue waiting (contrast with plain synchronized+wait(), which is also interruptible actually — that part behaves the same). 
  - The real gain here versus synchronized isn't interruption specifically (both support it for wait()), it's the combination of interruption + timeouts + multiple independent queues in one package.

```java
public interface Condition {
    void await() throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    void awaitUninterruptibly();
    boolean awaitUntil(Date deadline) throws InterruptedException;
    void signal();
    void signalAll();
}
```

### Separate conditions
```java
class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    BoundedBuffer(int capacity) { this.capacity = capacity; }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await(); // release lock, wait for "space" signal
            }
            queue.add(item);
            notEmpty.signal(); // tell exactly one consumer: "an item is here"
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await(); // release lock, wait for "item" signal
            }
            T item = queue.poll();
            notFull.signal(); // tell exactly one producer: "space opened up"
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

- Two entirely separate wait queues (notFull, notEmpty) share one lock. A put() only ever wakes threads parked on notEmpty; a take() only ever wakes threads parked on notFull. 
- No wasted wakeups, no ambiguity — this is the precision that plain synchronized/wait/notify cannot express.


## ReadWriteLock
- `ReadWriteLock` is a lock that splits access into two modes — read and write. A ReadWriteLock allows multiple threads to read a resource concurrently, but only one thread to write to it at a time. 
- This is useful when reads are frequent and writes are rare, as it allows for better concurrency than a simple exclusive lock.
- `ReadWriteLock` provides two lock objects backed by shared state:
```java
public interface ReadWriteLock {
    Lock readLock();
    Lock writeLock();
} 
```

| Operation                | Read lock held by others? | Write lock held by others? |
| ------------------------ | ------------------------- | -------------------------- |
| **Acquiring read lock**  | ✅ Allowed (shared)        | ❌ Blocks                   |
| **Acquiring write lock** | ❌ Blocks                  | ❌ Blocks                   |

In short:
- Multiple readers can hold the read lock at the same time — no blocking between them. 
- A writer needs full exclusivity — no readers and no other writer can be active. 
- Readers and a writer can never overlap.

Example:
```java
import java.util.concurrent.locks.ReentrantReadWriteLock;
import java.util.concurrent.locks.Lock;

ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();
Lock writeLock = rwLock.writeLock();
```

```java
class Cache {
    private final Map<String, String> data = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    public String get(String key) {
        readLock.lock();
        try {
            return data.get(key); // many threads can be in here at once
        } finally {
            readLock.unlock();
        }
    }

    public void put(String key, String value) {
        writeLock.lock();
        try {
            data.put(key, value); // exclusive — no readers or other writers
        } finally {
            writeLock.unlock();
        }
    }
}
```

**Reentrancy in ReentrantReadWriteLock**:
- ReentrantReadWriteLock is reentrant for both read and write locks. A thread holding the write lock can acquire it again, and a thread holding the read lock can acquire it again. 
- However, a thread holding the read lock cannot upgrade to the write lock without first releasing the read lock, as this would lead to potential deadlocks.
- A thread holding the write lock is allowed to acquire the read lock before releasing the write lock. This is called downgrading, and it's explicitly supported.

```java
writeLock.lock();
try {
    // make some update
    data.put("key", "value");

    readLock.lock(); // acquire read lock WHILE still holding write lock
    try {
        writeLock.unlock(); // release write lock — now holding only the read lock
        // continue reading with the guarantee no one else's write is interleaved
        return data.get("key");
    } finally {
        readLock.unlock();
    }
} finally {
    // writeLock already released above; this would be a no-op / already unlocked
}
```
**Fairness**

```java
ReentrantReadWriteLock fairRwLock = new ReentrantReadWriteLock(true);
```

- Unfair (default): favors throughput; can allow barging, and can also starve writers — if readers keep arriving in overlapping succession, a waiting writer might never find a moment when no readers hold the lock. 
- Fair: roughly FIFO ordering — a waiting writer will eventually get priority ahead of readers that arrived after it, preventing writer starvation, at some throughput cost.

**Condition support — write lock only**
- writeLock() supports newCondition(), just like ReentrantLock:

```Condition condition = writeLock.newCondition();```
- The read lock does not support Conditions — calling `readLock().newCondition()` throws `UnsupportedOperationException`.

**When it's actually worth using**
- ReadWriteLock adds overhead compared to a plain ReentrantLock — tracking reader counts, managing two lock modes, etc. It pays off specifically when:
  - Reads significantly outnumber writes. 
  - Read operations are non-trivial in duration 
  - Contention is real — under low contention, a simple ReentrantLock might actually outperform it, since the read-write bookkeeping isn't free.

### StampedLock
- `StampedLock` is a more modern lock (added in Java 8) that adds a third mode on top of read/write, optimistic reading.
- It's built for the common case where reads vastly outnumber writes and actual read-write conflicts are rare - letting readers proceed without blocking anything at all, then cheaply detect afterward if a write snuck in.

| Mode                   | Blocks writers?                                            | Blocks other readers?    | Blocks itself on writes?     |
| ---------------------- | ---------------------------------------------------------- | ------------------------ | ---------------------------- |
| **Write**              | Yes (exclusive)                                            | Yes                      | —                            |
| **Read (pessimistic)** | Yes, while held                                            | No — multiple readers OK | Blocks if a writer holds it  |
| **Optimistic read**    | No — doesn't block anything, and isn't blocked by anything | No                       | N/A — no actual lock is held |

- An optimistic read doesn't acquire a lock in the traditional sense at all — it just takes a "stamp" (a version number), reads the data, then checks afterward whether a write happened in between.
- If nothing changed, you're done — with zero blocking, zero CAS overhead on shared reader-count state, nothing.

**The stamp — central concept**
- Every StampedLock operation (read, write, or optimistic) returns a long "stamp" that represents the state of the lock at the moment you acquired it.


Step by step:
```java
class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead(); // does NOT block, does NOT acquire anything
        double currentX = x, currentY = y;    // read the data speculatively

        if (!sl.validate(stamp)) {
            // a write happened during our read — fall back to a real read lock
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }

        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```
- tryOptimisticRead() returns a stamp representing the lock's current version — but grants no actual lock. A writer could come in and mutate x/y at literally any moment while you're reading them. 
- You read the fields anyway, into local variables. 
- You call validate(stamp) — this checks whether any write lock was acquired (and released) between step 1 and now. If no write occurred, validate() returns true and your locally-read values are guaranteed consistent. 
- If a write did occur, validate() returns false — meaning your reads of x and y might be torn/inconsistent (e.g., you got the old x but the new y). You must discard them and retry, typically by falling back to a real, blocking readLock().

### What locking costs?
Locking costs you across several dimensions:

**1. Performance (even uncontended)**
Every lock acquire/release involves memory barriers and CAS instructions, plus reduced JIT optimization freedom around the locked region. This cost exists even when no other thread wants the lock.

**2. Contention cost (the big one)**
When threads actually compete, blocked threads get **parked** and later **woken** — OS-level context switches costing microseconds, far more than the lock operation itself. Heavy contention can mean more time spent switching than doing real work.

**3. Scalability (Amdahl's Law)**
Any code under an exclusive lock is serial by definition. That serial fraction caps your max speedup no matter how many cores you add. (This is why `ReadWriteLock` and `StampedLock` exist — to shrink the serial portion.)

**4. Correctness risk**
Locking introduces entire bug classes: **deadlock** (circular waiting), **livelock** (threads yield to each other endlessly), **starvation** (a thread never wins the race), **missed unlocks** (forgotten `finally`), and **lost wakeups** (wrong `Condition`, missing `while` loop). These bugs are non-deterministic and hard to reproduce.

**5. Composability**
Two independently thread-safe components don't combine into one atomic operation for free — you need a shared lock or a careful protocol, which reintroduces deadlock risk.

**6. Cache effects — false sharing**
Unrelated locked data sitting on the same CPU cache line causes invisible cross-core cache invalidation traffic, silently hurting performance with no logical contention at all.

**7. Human cost**
Concurrent code is hard to test (infinite interleavings), hard to review (invariants span lock boundaries), and produces "heisenbugs" that vanish under a debugger.

**Bottom line:** Every technique we covered — reentrancy, `tryLock`, `Condition`s, `ReadWriteLock`, `StampedLock`'s optimistic reads — exists to reduce one or more of these costs, not eliminate them. The engineering discipline is: **shrink critical sections, pick the cheapest lock that satisfies correctness, and accept that some cost is irreducible** — since the alternative to "locking costs something" isn't "free," it's "broken."

---
## Miscellaneous 
### where does lock actually live
- The monitor isn't a Java-visible field — it's a JVM-internal structure (JLS §17.1). You never see it as object.lock
- In the bytecode — `synchronized` compiles to explicit `monitorenter`/`monitorexit` instructions:
```java
void increment() {
    synchronized (this) {
        counter++;
    }
} 
```

```
$ javap -c MyClass
  0: aload_0
  1: dup
  2: astore_1
  3: monitorenter        // acquire this object's monitor
  4: aload_0
  5: dup
  6: getfield      #2    // Field counter:I
  9: iconst_1
 10: iadd
 11: putfield      #2
 14: aload_1
 15: monitorexit         // release monitor
 16: goto 24
 19: astore_2
 20: aload_1
 21: monitorexit         // exception path — must release too
 22: aload_2
 23: athrow
 24: return
```

- Notice there are two `monitorexit`s — one for normal exit, one in an exception handler. The compiler guarantees the lock is released even if the body throws.


### Why "per invariant" and not "per object" or "per variable"
Naively you might think: "one lock guards one field" or "one lock guards one object." Neither is right.

**Case A — one object, but the invariant spans multiple fields → they must share the same lock:**
```java
class NumberRange {
    private final Object lock = new Object();
    private int lower = 0;
    private int upper = 0;   // INVARIANT: lower <= upper, always

    public void setLower(int i) {
        synchronized (lock) {
            if (i > upper) throw new IllegalArgumentException();
            lower = i;
        }
    }

    public void setUpper(int i) {
        synchronized (lock) {
            if (i < lower) throw new IllegalArgumentException();
            upper = i;
        }
    }
}
```

If you guarded lower with one lock and upper with a different lock ("one lock per field"), a thread could observe lower > upper mid-update — a classic check-then-act race that violates the invariant even though each field is individually "safe." The invariant demands one lock covering both fields, not two locks covering one field each.

**Case B — the invariant spans multiple objects → a single object's own monitor isn't enough:**
```java
class Account {
    private final Object lock;   // shared lock, not "this"
    private long balance;

    Account(Object sharedLock, long balance) {
        this.lock = sharedLock;
        this.balance = balance;
    }

    static void transfer(Account from, Account to, long amt) {
        Object lock = from.lock; // both accounts share one lock object
        synchronized (lock) {
            if (from.balance < amt) throw new IllegalStateException();
            from.balance -= amt;
            to.balance   += amt;
        }
    }
}
```

Here the invariant is "`from.balance + to.balance` is conserved across the transfer." Locking `from` alone (`synchronized(from)`) protects from's own state but says nothing about `to`, and another thread could concurrently mutate `to` mid-transfer. "One lock per object" (each account synchronizing on itself) is actually the wrong granularity — you need one lock that covers the whole invariant, shared across both objects.

**And the converse also happens — one object, but multiple independent invariants, which can safely use separate locks for better concurrency:**
```java
class Server {
    private final Object connLock = new Object();
    private int activeConnections;      // invariant #1: >= 0

    private final Object statsLock = new Object();
    private long requestsServed;        // invariant #2: monotonic counter

    // connLock and statsLock protect unrelated invariants,
    // so using separate locks (instead of one "object lock")
    // lets updates to connections and stats proceed concurrently
}
```

If these two invariants were unrelated, forcing everything through a single synchronized(this) ("one lock per object") would serialize operations that don't need to be serialized — hurting concurrency for no correctness benefit.
