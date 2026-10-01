# Virtual Threads and Structured Concurrency
These are two related but distinct Java features (from Project Loom) aimed at making concurrent programming simpler and more scalable.

## Virtual Threads
### The problem they solve 
- Traditional Java threads (Thread) map 1:1 to OS threads. OS threads are expensive:
  - Each one takes significant memory (default ~1MB stack)
  - Creating/destroying them is costly 
  - Context switching between them costs CPU time 
  - You can typically only run a few thousand concurrently before things fall apart
- Because of this, high-throughput servers avoid "one thread per request" and instead use thread pools plus asynchronous/reactive programming (CompletableFuture, reactive streams, callbacks).
- This works, but it's notoriously hard to read, debug, and reason about — stack traces get mangled, and simple logic turns into a maze of chained callbacks.

### The idea
- Virtual threads are lightweight threads that are managed by the Java runtime rather than the OS. 
- They allow you to write code in a synchronous style (blocking calls) while still being able to handle thousands or millions of concurrent tasks.
- They're still `java.lang.Thread` objects — same API you already know.
- When a virtual thread performs a blocking operation (like I/O), the JVM automatically unmounts it from its underlying OS "carrier" thread, freeing that carrier to run other virtual threads. 
- When the blocking call completes, the virtual thread is remounted on some carrier thread to continue.
- This means you can write simple, blocking, sequential-looking code — and get the scalability of async code for free.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1)); // "blocks" the virtual thread, not an OS thread
            return fetchData();
        });
    }
} // waits for all tasks on close
```

- Try spawning 100,000 platform threads that each sleep a second — you'll likely crash the JVM. With virtual threads, this is trivial, because sleeping virtual threads don't tie up OS threads.

### Creation — Normal Thread vs Virtual Thread
| Type                         | Creation                                                              | Package                                     |
|------------------------------|-----------------------------------------------------------------------|---------------------------------------------|
| **Platform (normal) thread** | `new Thread(runnable)` or `Thread.ofPlatform()`                       | `java.lang.Thread`                          |
| **Virtual thread**           | `Thread.ofVirtual()` or `Executors.newVirtualThreadPerTaskExecutor()` | `java.lang.Thread` / `java.util.concurrent` |

- Both come from the same package `java.lang.Thread`

```java
// Platform thread — old way
Thread t1 = new Thread(() -> System.out.println("platform"));
t1.start();

// Platform thread — new builder API (JDK 19+)
Thread t2 = Thread.ofPlatform().unstarted(() -> System.out.println("platform"));
t2.start();

// Virtual thread — builder API
Thread t3 = Thread.ofVirtual().unstarted(() -> System.out.println("virtual"));
t3.start();

// Virtual thread — convenience factory
Thread t4 = Thread.startVirtualThread(() -> System.out.println("virtual, auto-started"));

// Virtual thread — via executor (most common in practice)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> System.out.println("virtual, via executor"));
}
```

### What a virtual thread actually is 
- A virtual thread is a Thread instance whose execution is scheduled by the JVM, not by the OS.
- It's an object implementing java.lang.Thread (technically backed by an internal VirtualThread class, but you never touch that directly — you only ever see Thread).
- The JVM maintains a pool of OS threads (carrier threads) that actually run the virtual threads. When a virtual thread blocks, the JVM can unmount it from its carrier and let that carrier run other virtual threads.
- A virtual thread is mounted onto a carrier thread only while it's actually doing work. When it blocks (I/O, sleep, lock wait), it's unmounted, and the carrier is freed to run a different virtual thread.
- The virtual thread's stack is not a fixed-size OS stack; it's a dynamically growing structure managed by the JVM. This allows you to have many more virtual threads than OS threads.

### When to use what? 
| Use **platform threads** when                                                      | Use **virtual threads** when                                                   |
|------------------------------------------------------------------------------------|--------------------------------------------------------------------------------|
| CPU-bound / computation-heavy work (matrix math, image processing, parsing)        | I/O-bound work: HTTP calls, DB queries, file reads, waiting on network sockets |
| You need a small, fixed, long-lived pool doing continuous work                     | You need high concurrency — thousands/millions of tasks, each mostly *waiting* |
| Long-running background/daemon workers tied to system resources                    | Per-request handlers in a server (one virtual thread per incoming request)     |
| Code relies on thread identity/affinity for pinning to specific hardware resources | You want simple, blocking-style code instead of async chains                   |


### `synchronized` and Virtual Threads — Pinning
- Virtual threads can be blocked on `synchronized` locks just like platform threads.
- When a virtual thread enters a synchronized block/method (pre-JDK 24) and then calls something blocking inside it:
  - The JVM cannot unmount the virtual thread at that point. Monitor locks (synchronized) are implemented at a level tied to the OS thread's identity/state — the JVM's continuation-freezing mechanism couldn't safely detach the virtual thread from its carrier while holding that lock's low-level state.
  - Result: the virtual thread stays pinned to its carrier thread for the entire duration of the blocking call inside the synchronized block.
  - The carrier thread is now stuck waiting too — it can't go serve other virtual threads. If many virtual threads pin their carriers simultaneously, you can exhaust the whole carrier pool, causing everything to stall — exactly the scalability problem virtual threads were meant to solve, reintroduced.
  - The fix historically recommended: replace `synchronized` with `java.util.concurrent.locks.ReentrantLock`, which is virtual-thread-aware and allows proper unmounting while waiting/blocked.

```java
// Problematic pre-JDK24 pattern with virtual threads:
synchronized (lock) {
    result = blockingNetworkCall(); // pins the carrier thread for the whole call!
}

// Solution
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    result = blockingNetworkCall(); // does NOT pin — can unmount normally
} finally {
    lock.unlock();
}
```

## Structured Concurrency
### The problem it solves
- Once you can spawn huge numbers of threads cheaply, a new problem appears: managing their lifecycles safely.
- Unstructured concurrency — where you fire off threads/tasks that outlive the method that created them — leads to:
  - Leaked threads (nobody waits for them or cancels them)
  - Cancellation that doesn't propagate (if one subtask fails, siblings keep running)
  - Error handling scattered across callbacks 
  - Hard-to-read code where the "shape" of concurrent execution doesn't match the code's lexical structure

### The idea 
- Structured concurrency is a programming model that treats groups of threads/tasks as a single unit of work, with a clear lifecycle. Tied to the lifetime of a block of code — much like structured programming ties control flow to lexical blocks.
- You create a "scope" for concurrent tasks, and when that scope ends (normally or exceptionally), all tasks in it are completed or cancelled.
- If a task spawns child tasks, the parent doesn't return until all children complete, fail, or are cancelled together.
- This makes it easier to reason about concurrency, because the structure of your code reflects the structure of your concurrent execution.
- In Java, structured concurrency is implemented via the `StructuredTaskScope` API. You can create a scope, submit tasks to it, and then wait for all tasks to complete or handle exceptions in a unified way.

```java
try (var scope = StructuredTaskScope.open(
        StructuredTaskScope.Joiner.<String>allSuccessfulOrThrow())) {

    Supplier<String> user   = scope.fork(() -> fetchUser());
    Supplier<String> orders = scope.fork(() -> fetchOrders());

    scope.join();           // wait for both, propagate cancellation/errors together

    return combine(user.get(), orders.get());
} // scope closes: any still-running forks are cancelled automatically
```

Key guarantees:
- No thread/task leaks — the try-with-resources block ensures nothing outlives its scope. 
- Error propagation — if one subtask fails, others can be automatically cancelled (depending on the "joiner" policy, e.g. fail-fast vs. wait-for-all). 
- Cancellation is automatic and consistent — leaving the block always cleans up. 
- Readable structure — the nesting of your code visually reflects the nesting of concurrent tasks, just like sequential code.

> **You use them together**: virtual threads make it cheap to spawn a thread per subtask, and structured concurrency makes it safe to manage the resulting swarm of threads without leaks or dangling cancellation. Structured concurrency doesn't require virtual threads, but they're a natural pairing — you can freely fork many subtasks without worrying about OS thread exhaustion.
