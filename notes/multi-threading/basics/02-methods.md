## Methods related to multi-threading and threads 
### Methods on Thread itself

- **`start()`** — begins a new thread of execution; the JVM calls `run()` on a new call stack. You never call `run()` directly if you want actual concurrency.
- **`run()`** — contains the code that executes in the new thread. If called directly (not via `start()`), it just runs on the current thread.
- **`join()`** — makes the calling thread wait until the thread it's called on finishes. Overloads accept a timeout (`join(long millis)`). Lets say main calls `a.join(); b.join(); /\*next statements\*/`— main will wait for a to finish, then b, then continue with next statements.
- **`sleep(long millis)`** — static method; pauses the *current* thread for a given time, without releasing any locks it holds.
- **`interrupt()`** — signals a thread to stop what it's doing. Depending on the thread's state we have 
  - if it's sleeping/waiting, it throws `InterruptedException` and clears the flag.
  - if it's running, it sets the interrupt flag; the thread can check it via `isInterrupted()` and decide to exit gracefully. see misc. for more details.
- **`isInterrupted()` / `interrupted()`** — check interrupt status (instance vs. static, the latter also clears the flag).
- **`setPriority()` / `getPriority()`** — hints to the scheduler about relative importance.
- **`setDaemon(boolean)` / `isDaemon()`** — marks a thread as a background/daemon thread that won't prevent JVM shutdown.
- **`getName()` / `setName()`**, **`getId()`**, **`currentThread()`** (static) — identity/utility methods.
- **`yield()`** — static hint to the scheduler that the current thread is willing to let others run; not guaranteed to do anything.

## Methods on `Object` (used for synchronization)

These live on `Object`, not `Thread`, because any object can serve as a lock/monitor:

- **`wait()`** — releases the object's monitor lock and suspends the current thread until notified (or timeout, if given). Must be called inside a `synchronized` block on that object.
- **`wait(long timeout)` / `wait(long timeout, int nanos)`** — bounded wait variants.
- **`notify()`** — wakes up *one* arbitrary thread waiting on this object's monitor.
- **`notifyAll()`** — wakes up *all* threads waiting on this object's monitor; they then compete to reacquire the lock.

**Important nuance:** `wait`/`notify`/`notifyAll` must be called from within a `synchronized` block/method on the same object, or you get `IllegalMonitorStateException`. This is the classic producer-consumer pattern building block.

## The `synchronized` keyword and locks

- **`synchronized` blocks/methods** — enforce mutual exclusion; only one thread can hold a given object's monitor at a time.
- **`Lock` interface** (`java.util.concurrent.locks`) — more flexible alternative: `lock()`, `unlock()`, `tryLock()`, `lockInterruptibly()`.
- **`ReentrantLock`**, **`ReadWriteLock`** — concrete lock implementations with more control than built-in `synchronized`.
- **`Condition`** (from `Lock.newCondition()`) — the `Lock`-based analog of `wait`/`notify`: `await()`, `signal()`, `signalAll()`.

## Higher-level concurrency utilities (`java.util.concurrent`)

Since raw `wait`/`notify` is error-prone, modern code often uses:

- **`ExecutorService`** — manages a pool of threads; `submit()`, `execute()`, `shutdown()`, `awaitTermination()`.
- **`Future` / `CompletableFuture`** — represent results of asynchronous computation; `get()`, `isDone()`, `cancel()`, `thenApply()`, etc.
- **`CountDownLatch`** — `countDown()`, `await()` — lets threads wait until a set of operations completes.
- **`CyclicBarrier`** — `await()` — synchronizes threads at a common barrier point.
- **`Semaphore`** — `acquire()`, `release()` — controls access to a limited number of permits.
- **`AtomicInteger`, `AtomicLong`, `AtomicReference`**, etc. — lock-free atomic operations: `get()`, `set()`, `incrementAndGet()`, `compareAndSet()`.
- **`BlockingQueue`** — `put()`, `take()` — thread-safe queue that blocks when full/empty, often replacing manual wait/notify producer-consumer code.

---

## Miscellaneous 
## `interrupt()` in Java

`interrupt()` doesn't forcibly stop a thread. It just **sets an internal flag** on the target thread ("interrupted status = true") and, if that thread happens to be blocked in `sleep()`, `wait()`, or `join()`, it **wakes it up immediately** by throwing an `InterruptedException`.

Think of it as *politely tapping a thread on the shoulder* — the thread has to check for that tap and decide what to do. If the thread's code never checks, `interrupt()` has no real effect on its control flow.

### Two situations

**1. Thread is blocked (sleeping/waiting/joining)**
The blocking call throws `InterruptedException` right away, and the interrupted status is cleared automatically when the exception is thrown.

**2. Thread is doing normal computation (a loop, etc.)**
Nothing happens automatically — the flag is just set. The thread must periodically check `Thread.isInterrupted()` (or the static `Thread.interrupted()`) itself and respond.

### Example 1 — interrupting a sleeping thread

```java
public class InterruptSleepExample {
    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            try {
                System.out.println("Worker: going to sleep for 10 seconds...");
                Thread.sleep(10_000);
                System.out.println("Worker: woke up normally"); // won't reach here
            } catch (InterruptedException e) {
                System.out.println("Worker: I was interrupted while sleeping!");
            }
        });

        worker.start();
        Thread.sleep(1000);      // let worker fall asleep first
        worker.interrupt();      // wake it up early
    }
}
```

**Output:**
```
Worker: going to sleep for 10 seconds...
Worker: I was interrupted while sleeping!
```

The `sleep()` call immediately throws `InterruptedException` instead of waiting the full 10 seconds.

### Example 2 — interrupting a busy loop (no blocking call)

Here the thread must check the flag itself, since there's nothing to "wake up" from:

```java
public class InterruptLoopExample {
    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            int count = 0;
            while (!Thread.currentThread().isInterrupted()) {
                count++;
                // simulate work
            }
            System.out.println("Worker: exiting loop, counted up to " + count);
        });

        worker.start();
        Thread.sleep(1000);   // let it run for a second
        worker.interrupt();   // just sets the flag; loop checks it and exits
    }
}
```

If the loop never called `isInterrupted()`, calling `interrupt()` would do nothing visible — the thread would spin forever, since no exception is thrown for non-blocking code.

### Key points to remember

- `interrupt()` sets a flag; it doesn't kill the thread.
- Blocking methods (`sleep`, `wait`, `join`) react to it by throwing `InterruptedException`.
- Non-blocking code must **cooperatively check** `isInterrupted()`.
- Catching `InterruptedException` clears the flag — if you swallow it without handling, callers further up the stack lose the information that an interrupt happened. Best practice is to either handle it there or re-set the flag / rethrow:

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restore the interrupted status
}
```

This is why `interrupt()` is described as a **cooperative cancellation mechanism** rather than a forced stop — it depends on the target thread's code being written to respect it.

---

## `wait()`, `notify()`, and `notifyAll()`

These are methods on `Object` (not `Thread`) used for **inter-thread communication** — letting one thread pause until another thread tells it that some condition has changed.

They must always be called from inside a `synchronized` block/method on the object you're calling them on — otherwise you get `IllegalMonitorStateException`.

### What each one does

- **`wait()`** — the current thread releases the lock it holds on this object and goes to sleep, sitting in that object's "wait set." It stays there until another thread calls `notify()`/`notifyAll()` on the same object (or a timeout elapses, if you used the timed overload).
- **`notify()`** — wakes up **one** arbitrary thread waiting on this object's monitor. That thread doesn't resume immediately — it has to reacquire the lock first, so it competes with any other threads trying to enter synchronized blocks on that object.
- **`notifyAll()`** — wakes up **all** threads waiting on this object's monitor; they all then compete to reacquire the lock, but only one gets it at a time.

**Why release the lock in `wait()`?** If it didn't, no other thread could ever get in to call `notify()` — you'd deadlock immediately.

### Classic example: Producer-Consumer with a bounded buffer

```java
import java.util.LinkedList;
import java.util.Queue;

public class ProducerConsumerExample {
    public static void main(String[] args) {
        SharedBuffer buffer = new SharedBuffer(5);

        Thread producer = new Thread(() -> {
            int value = 0;
            while (true) {
                buffer.produce(value++);
                try { Thread.sleep(300); } catch (InterruptedException e) {}
            }
        });

        Thread consumer = new Thread(() -> {
            while (true) {
                buffer.consume();
                try { Thread.sleep(1000); } catch (InterruptedException e) {}
            }
        });

        producer.start();
        consumer.start();
    }
}

class SharedBuffer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity;

    public SharedBuffer(int capacity) {
        this.capacity = capacity;
    }

    public synchronized void produce(int value) {
        while (queue.size() == capacity) {
            try {
                System.out.println("Buffer full. Producer waiting...");
                wait();  // releases lock, sleeps until notified
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        queue.add(value);
        System.out.println("Produced: " + value + " | Buffer: " + queue);
        notifyAll();  // wake up consumer(s) waiting for items
    }

    public synchronized void consume() {
        while (queue.isEmpty()) {
            try {
                System.out.println("Buffer empty. Consumer waiting...");
                wait();  // releases lock, sleeps until notified
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        int value = queue.poll();
        System.out.println("Consumed: " + value + " | Buffer: " + queue);
        notifyAll();  // wake up producer(s) waiting for space
    }
}
```

### Sample output pattern

```
Produced: 0 | Buffer: [0]
Produced: 1 | Buffer: [0, 1]
Consumed: 0 | Buffer: [1]
Produced: 2 | Buffer: [1, 2]
...
Buffer full. Producer waiting...
Consumed: 4 | Buffer: [2, 3]
Produced: 5 | Buffer: [2, 3, 5]
```

### Key points

1. **Always call `wait()` in a loop**, checking the condition (`while (queue.size() == capacity)`), not an `if`. This guards against:
  - **Spurious wakeups** — the JVM is allowed to wake a waiting thread without an actual `notify()`.
  - **Multiple waiters** — after `notifyAll()`, several threads wake up and race for the lock; by the time a given thread gets it, the condition might already be false again (e.g., someone else already consumed the last item).

2. **`notify()` vs `notifyAll()`**:
  - `notify()` is more efficient but risky — if you have multiple different conditions or multiple types of waiting threads, you might wake the "wrong" one and cause a **lost wakeup** (a thread waits forever because the wakeup went to a thread that couldn't proceed anyway).
  - `notifyAll()` is safer in general and is what most textbook examples use. Use `notify()` only when you're certain every waiting thread is waiting for the exact same condition and any one of them can proceed.

3. `wait()`/`notify()` operate on the monitor of **the object you call them on** — in this example, `this` (the `SharedBuffer` instance), because `produce`/`consume` are `synchronized` instance methods.

4. This whole pattern is essentially what `java.util.concurrent.BlockingQueue` does for you internally — in real production code, you'd typically just use `LinkedBlockingQueue` with `put()`/`take()` instead of hand-rolling `wait`/`notify`. It's still important to understand though, since it's the foundation the higher-level utilities are built on.

Want me to show the same producer-consumer logic rewritten using `Lock`/`Condition`, or using `BlockingQueue` for comparison?