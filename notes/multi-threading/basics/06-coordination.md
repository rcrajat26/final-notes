# Coordination - waiting for each other 
## Thread dependency 
- A thread is dependent on another task to complete for example a payment service provider to respond to complete the payment.
- An obvious way is to keep polling the PSP until it responds:
```java
while (queue.isEmpty()) {
    // keep checking
}
process(queue.poll()); 
```

- **It burns a whole CPU doing nothing**: A spinning thread is indistinguishable from a thread doing real work as far as the scheduler is concerned. On a 4-core box, four spinning consumers consume the machine. 
- **It may never see the change at all**: The JIT can hoist the check out of the loop and spin on a stale register value forever.
- **Adding a sleep makes it wrong differently**: Thread.sleep(10) inside the loop fixes the CPU burn and introduces up to 10 ms of latency on every single item. For a price feed processing thousands of ticks a second, that is fatal. 
- What you actually want is for the thread to stop consuming CPU entirely, and for the OS to wake it the instant the condition changes. That requires help from the runtime.

## Wait/Notify
- Every Java object has a monitor. It also has a wait set — a list of threads parked on that object.
```java
synchronized (lock) {
    while (!conditionIsTrue()) {
        lock.wait();            // park; release the monitor; wake on notify
    }
    // condition is true AND we hold the monitor
}

synchronized (lock) {
    changeSomething();
    lock.notifyAll();           // wake threads in the wait set
} 
```

### The five rules
- **1. You must hold the monitor to call**: `wait()`, `notify()`, and `notifyAll()` are methods on an `object` they only make sense inside a synchronized block, which holds a lock on that given `object`. If we don't hold the lock on that `object`, we get an `IllegalMonitorStateException`. 
- **2. `wait()` releases the monitor while parked, and reacquires it before returning**: Unlike Thread.sleep(), which holds every lock it owns for the whole duration. This is exactly what lets another thread do `synchronized(lock) { ...change state...; lock.notify(); }` while the thread is waiting.
- **3. `wait()` condition must be tested in a loop**: The thread may wake up without the condition being true. This is called a spurious wakeup. So we must check the condition in a loop, and call wait() again if it is not true. Further, notify() (or notifyAll()) wakes a thread, but by the time it reacquires the lock, some other thread may have already run and consumed whatever condition was made true.
- **4. `notify()` wakes one arbitrary waiter; `notifyAll()` wakes all of them**: `notify()` picks one thread from the wait set — you don't get to choose which, and it's not necessarily FIFO. `notifyAll()` wakes every thread waiting on that monitor; they'll all re-contend for the lock, one at a time, each re-checking its while condition. Further, If different threads are waiting for different conditions on the same lock, notify() can wake the "wrong" one, which then rechecks its while, finds its condition still false, and goes back to waiting. Hence, notifyAll() is the safe default.
- **5. Signaling before waiting is lost forever**: notify signals don't queue up. If no thread is currently in wait() when you call notify(), the signal just evaporates — there's no "memory" of it for a thread that calls wait() a moment later.

### `notify()` can deadlock your program
**A bounded buffer for the price feed**
- A bounded buffer is a queue with a maximum capacity. If the queue is full, producers must wait until there is space. If the queue is empty, consumers must wait until there is an item to consume.
- For ex: The market data feed produces ticks; the matching engine consumes them. The buffer between them must be bounded — an unbounded one just relocates the problem into an out-of-memory error.
```java
class TickBuffer {
    private final Deque<Tick> q = new ArrayDeque<>();
    private final int capacity;

    synchronized void put(Tick t) throws InterruptedException {
        while (q.size() == capacity) wait();     // full: producer waits
        q.addLast(t);
        notify();                                 // <-- the bug
    }

    synchronized Tick take() throws InterruptedException {
        while (q.isEmpty()) wait();               // empty: consumer waits
        Tick t = q.removeFirst();
        notify();                                 // <-- the bug
        return t;
    }
}
```
- This looks symmetrical and reasonable. It has a fatal flaw: producers and consumers wait in the same wait set, and notify() picks an arbitrary member of it.
- Both put() and take() park their threads in the same wait set on the same monitor. When notify() fires, it wakes one arbitrary thread — which might be the wrong type. 

| Step         | Event                                                         | Waiting threads    |
| ------------ | ------------------------------------------------------------- | ------------------ |
| 1            | Buffer empty                                                  | C1, C2, C3         |
| 2            | P1: `put(tick)` → `notify()` wakes C1                         | C2, C3, P2, P3     |
| 3            | P2, P3: try `put()`, buffer full → `wait()`                   | C2, C3, P1, P2, P3 |
| 4            | C1: takes item, calls `notify()` → wakes C2 (not a producer!) | C3, P1, P2, P3, C1 |
| 5            | C2: wakes, checks `isEmpty()` → `true`, calls `wait()`        | All 6 threads      |
| **DEADLOCK** | No thread is running. No more `notify()` can fire.            | —                  |
 
Reason for failure:
-  Signal wakes the wrong type: P1 produces and signals hoping to wake a consumer, but notify() picks another producer P2 instead. P2 re-checks, finds buffer full, and waits — consuming the only wakeup signal.
- No recovery path: Once a producer or consumer wakes and re-checks its condition and waits again, the signal is gone forever. The next signal might wake the wrong type again. 

Solution:
> Use notifyAll() unless you can prove every thread in the wait set is waiting for the identical condition. notify() is a micro-optimisation that trades a correct program for a slightly cheaper one. The measured difference above is between working and permanently hung.

Note:
The cost of notifyAll() is real but modest: n threads wake, one proceeds, the rest re-check and go back to sleep. That is called the thundering herd, and with a large wait set it wastes context switches. The right fix is not notify() — it is to stop mixing different kinds of waiter in one wait set.

## `Condition` — one wait set per condition
- A monitor has exactly one wait set. A ReentrantLock can have as many as you like.
```java
class TickBuffer {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Deque<Tick> q = new ArrayDeque<>();
    private final int capacity;

    void put(Tick t) throws InterruptedException {
        lock.lock();
        try {
            while (q.size() == capacity) notFull.await();
            q.addLast(t);
            notEmpty.signal();           // signal() is SAFE here
        } finally { lock.unlock(); }
    }

    Tick take() throws InterruptedException {
        lock.lock();
        try {
            while (q.isEmpty()) notEmpty.await();
            Tick t = q.removeFirst();
            notFull.signal();            // also safe
            return t;
        } finally { lock.unlock(); }
    }
} 
```
- Here we have a single lock and two conditions hence we must use Reentrant lock, its not possible with synchronized.

Now `signal()` is correct, and this is the point worth extracting:
> `notify()` was never inherently wrong. It was wrong because one wait set held two kinds of waiter. Every thread on notFull is a producer and every thread on notEmpty is a consumer, so waking one arbitrary member of either set always wakes the right kind.

- Separating the conditions removes both the deadlock and the thundering herd.
- The while loop is still mandatory — spurious wakeups apply to `await()` exactly as they do to `wait()`


| Monitor | `Condition` |
|---|---|
| `synchronized (o)` | `lock.lock()` / `unlock()` in `finally` |
| `o.wait()` | `cond.await()` |
| `o.notify()` | `cond.signal()` |
| `o.notifyAll()` | `cond.signalAll()` |
| one wait set | as many as you need |
| not interruptible while blocked on entry | `lockInterruptibly()`, `awaitNanos`, `awaitUntil` |

In production you would not write either version — `ArrayBlockingQueue` (file 07) is exactly this class, correct and tuned. Building it once is how the JDK version stops being magic.

## CountDownLatch
- `CountDownLatch` is a one-shot synchronization gate: one or more threads wait until a counter hits zero, and other threads decrement that counter as they finish work

```java
CountDownLatch latch = new CountDownLatch(3);   // count starts at 3

latch.await();          // block until count reaches 0
latch.await(2, SECONDS); // block with a timeout, returns boolean
latch.countDown();      // decrement count by 1 (no-op if already 0)
latch.getCount();       // current count, mostly for debugging/tests
```

Example:
```java
CountDownLatch latch = new CountDownLatch(3);

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        doWork();
        latch.countDown();   // signal "I'm done"
    }).start();
}

latch.await();   // main thread blocks until all 3 have called countDown()
System.out.println("All workers finished");
```

- Any thread can call countDown() (workers finishing tasks), and any thread can call await() (one coordinator, or several — all of them release together once the count hits zero).

### Another way:
- Flip it around — instead of waiting for workers to finish, make workers wait for a "go" signal:
```java
CountDownLatch startGate = new CountDownLatch(1);

for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        startGate.await();   // all threads block here
        doWork();             // then all proceed together
    }).start();
}

Thread.sleep(1000);   // simulate setup
startGate.countDown(); // count 1 -> 0, releases everyone at once 
```
- This is the classic pattern for load tests or benchmarks: get N threads all parked at the gate, then release them simultaneously to maximize contention/concurrency at a precise moment.

- Internally, CountDownLatch is doing roughly what you'd build by hand in wait/notify:

```java
// conceptually, not the real implementation - in actual implementation it uses AbstractQueuedSynchronizer
synchronized (lock) {
    while (count > 0) {
        lock.wait();
    }
}
// countDown():
synchronized (lock) {
    count--;
    if (count == 0) {
        lock.notifyAll();
    }
}
```

**The critical property: it's one-shot**
- Once the count reaches 0, it stays at 0 forever. There's no reset(). countDown() after the count is already 0 does nothing. await() called after the count already hit 0 returns immediately — it doesn't block.
- This connects directly to rule 5 from before ("signaling before waiting is lost forever") — except CountDownLatch deliberately fixes that problem for you. Since the "signal" is a permanent state change (the count), not a transient notify, a thread that calls await() after the count already hit zero still sees the correct state and doesn't block.


## Semaphore 
- A `Semaphore` manages a fixed number of "permits." Threads acquire a permit before entering a limited resource, and release it when done.
- Unlike CountDownLatch, it's reusable — permits can be given back and taken again indefinitely.

```java
Semaphore sem = new Semaphore(3);        // 3 permits available
sem.acquire();       // blocks until a permit is available, then takes one
sem.release();       // gives a permit back
sem.tryAcquire();    // non-blocking, returns boolean immediately
sem.tryAcquire(2, SECONDS); // blocking with timeout
sem.availablePermits(); // current count, mostly for debugging
```

**The canonical use case: bounding concurrency**:
- Say you have a resource that can only safely handle 3 concurrent users — a connection pool, a rate limiter, a fixed set of parking spots.

```java
Semaphore pool = new Semaphore(3);

void useResource() throws InterruptedException {
    pool.acquire();          // take a permit (block if none free)
    try {
        doWorkWithLimitedResource();
    } finally {
        pool.release();       // always give it back, even on exception
    }
}
```
- Ten threads can call useResource() at once; only 3 will be inside doWorkWithLimitedResource() at any moment. The other 7 block at acquire() until a permit frees up.

Conceptually:
```java
// conceptually, not the real implementation. The real JDK implementation uses AbstractQueuedSynchronizer
synchronized (lock) {
    while (permits == 0) {
        lock.wait();
    }
    permits--;
}
// release():
synchronized (lock) {
    permits++;
    lock.notify();   // wake one waiter — a permit just became available
}
```

| Feature        | `Semaphore(1)`                                                                     | `synchronized` / `ReentrantLock`                          |
|----------------|------------------------------------------------------------------------------------|-----------------------------------------------------------|
| **Ownership**  | **None** — any thread can call `release()`, even one that never called `acquire()` | Strict — only the thread holding the lock can release it  |
| **Reentrant?** | No — if the same thread calls `acquire()` twice, it blocks itself forever          | Yes — the same thread can acquire the lock multiple times |

- By default, Semaphore does not guarantee FIFO ordering among blocked threads
- To introduce order use:
```java
Semaphore fairSem = new Semaphore(3, true);  // fair ordering
```
