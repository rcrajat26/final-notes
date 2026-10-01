# Liveness 
- Liveness is one of the two fundamental properties concurrent systems are judged on. 
  - Safety properties mean "nothing bad happens" (e.g., no data corruption). 
  - Liveness properties mean "something good eventually happens" — the system keeps making progress, and threads/processes don't get stuck forever.
- When liveness fails, you get one of three failure modes: deadlock, livelock, or starvation.

## Deadlock
**Definition**: Two or more threads are each waiting for a resource held by the other, and neither can proceed. Everyone is blocked, forever.

### The scenario
- Wallet-to-wallet transfer. The natural implementation locks both wallets so neither balance can change mid-transfer:
```java
void transfer(Wallet from, Wallet to, long amt) {
    synchronized (from) {
        synchronized (to) {
            from.balance -= amt;
            to.balance   += amt;
        }
    }
}
```

- Now imagine two threads trying to transfer money in opposite directions at the same time:
```
thread 1: transfer(walletA, walletB, 5000)    locks A, then wants B
thread 2: transfer(walletB, walletA, 3000)    locks B, then wants A
```
- Each holds what the other needs. Neither will ever release, because releasing only happens on exit and neither can exit.

### The four Coffman conditions
Deadlock requires all four of these simultaneously:

1. Mutual exclusion — a resource can be held by only one thread at a time. 
2. Hold and wait — a thread holds one resource while waiting for another. 
3. No preemption — a resource can't be forcibly taken away; it's released voluntarily. 
4. Circular wait — a cycle of threads exists, each waiting on the next.

**Classic example**:
```java
Object lockA = new Object();
Object lockB = new Object();

// Thread 1
synchronized (lockA) {
    Thread.sleep(50);
    synchronized (lockB) {   // waits for lockB
        // ...
    }
}

// Thread 2
synchronized (lockB) {
    Thread.sleep(50);
    synchronized (lockA) {   // waits for lockA
        // ...
    }
}
```

`ThreadMXBean.findDeadlockedThreads()` from inside the same JVM:
```
ThreadMXBean found 2 deadlocked threads:
  transfer-A-to-B    BLOCKED
      waiting to lock : Deadlock$Wallet@498d318c
      held by         : transfer-B-to-A
  transfer-B-to-A    BLOCKED
      waiting to lock : Deadlock$Wallet@333291e3
      held by         : transfer-A-to-B
```

### Reading the thread dump
- Below is the real output of `jcmd <pid> Thread.print` against that process.
- **The individual thread entry**:
```
"transfer-A-to-B" #13 [293] daemon prio=5 os_prio=0 cpu=8.60ms elapsed=6.28s
   java.lang.Thread.State: BLOCKED (on object monitor)
	at Deadlock.transfer(Deadlock.java:19)
	- waiting to lock <0x00000000c1619428> (a Deadlock$Wallet)
	- locked <0x00000000c16193e0> (a Deadlock$Wallet)
	at Deadlock.lambda$main$0(Deadlock.java:30)
```

Line by line, because every piece is load-bearing:
- "transfer-A-to-B" — the thread name. This is the payoff for naming your threads; the default pool-1-thread-7 would tell you nothing here.
- cpu=8.60ms elapsed=6.28s — it has been alive 6 seconds and used 8 ms of CPU. A thread that is stuck has near-zero CPU; a thread that is spinning has CPU time approaching its elapsed time. This one distinction separates deadlock from livelock at a glance.
- BLOCKED (on object monitor) — waiting to enter a synchronized block.
- waiting to lock <0x...c1619428> — the monitor it wants.
- locked <0x...c16193e0> — the monitor it already holds.

Thread dump for other thread:
```
"transfer-B-to-A" #14 [294] ... cpu=6.28ms elapsed=6.28s
   java.lang.Thread.State: BLOCKED (on object monitor)
	at Deadlock.transfer(Deadlock.java:19)
	- waiting to lock <0x00000000c16193e0> (a Deadlock$Wallet)
	- locked <0x00000000c1619428> (a Deadlock$Wallet)
```
- The addresses are swapped. A wants what B holds and B wants what A holds. That is the cycle, visible directly in the text.
- The JVM finds it for you:
```
Found one Java-level deadlock:
=============================
"transfer-A-to-B":
  waiting to lock monitor 0x00007f2644007ea0 (object 0x00000000c1619428, a Deadlock$Wallet),
  which is held by "transfer-B-to-A"

"transfer-B-to-A":
  waiting to lock monitor 0x00007f2650005e70 (object 0x00000000c16193e0, a Deadlock$Wallet),
  which is held by "transfer-A-to-B"

Found 1 deadlock.
```
- Search every thread dump for the word deadlock first. It costs nothing and sometimes ends the investigation immediately.


### How to take a dump

```bash
jcmd <pid> Thread.print      # preferred
jstack <pid>                 # older, equivalent
kill -3 <pid>                # SIGQUIT — dump goes to the process's stdout
```

`kill -3` is the one that works when nothing else can attach — it needs no tooling in the
container, but the output lands in stdout, so you need to know where that goes.

### The general reading method

| What you see                                       | What it means                                              |
|----------------------------------------------------|------------------------------------------------------------|
| `Found one Java-level deadlock`                    | done — read the cycle                                      |
| Many threads `BLOCKED` on the same monitor address | lock contention; find the one thread that `locked` it      |
| Many threads `RUNNABLE` deep in a socket read      | **not** CPU-bound — a slow downstream service (file 01 §9) |
| Many threads `WAITING (parking)` on a queue        | the pool is idle; the bottleneck is upstream               |
| Many threads in the same application frame         | that method is your hotspot                                |
| High `cpu=` relative to `elapsed=` while stuck     | spinning — livelock, not deadlock                          |

**Prevention strategies**
- Lock ordering: always acquire locks in a fixed global order (e.g., by object hash code or an assigned ID). This breaks circular wait.
- Lock timeouts: use tryLock(timeout) instead of blocking indefinitely; back off and retry if you can't get the lock.
- Avoid nested locks: acquire only one lock at a time where possible.
- Use higher-level concurrency utilities: java.util.concurrent collections/executors, or lock-free structures.
- Deadlock-avoidance algorithms: like the Banker's Algorithm — grants resource requests only if the resulting state is still "safe" (i.e., some order exists in which all processes could finish).

## Deadlock without any locks
### Thread pool / executor deadlock
- A fixed-size thread pool where a task submits another task to the same pool and blocks waiting for its result.
```java
ExecutorService pool = Executors.newFixedThreadPool(1); // only 1 thread!

Future<Integer> future = pool.submit(() -> {
    Future<Integer> inner = pool.submit(() -> 42); // needs a free thread
    return inner.get(); // blocks waiting for a thread that will never come
});

future.get(); // hangs forever
```
- The outer task is waiting for the inner task to complete, but the inner task can't start because the only thread in the pool is already busy with the outer task. This is a form of deadlock even though no explicit locks are used.
- **Prevention**: Avoid submitting tasks to the same thread pool from within a task that is already running in that pool. Use a larger pool size or a different executor for nested tasks.

### Bounded queue / channel deadlock (message passing)
- A producer-consumer scenario where the producer fills a bounded queue and the consumer is blocked waiting for items, but the producer is also blocked because the queue is full and it can't proceed.
```java
BlockingQueue<Integer> qA = new ArrayBlockingQueue<>(1);
BlockingQueue<Integer> qB = new ArrayBlockingQueue<>(1);

// Thread 1: put into qA, then take from qB
qA.put(1);
int x = qB.take(); // blocks — nobody will ever put into qB first

// Thread 2: put into qB, then take from qA
qB.put(2);
int y = qA.take(); // blocks — same problem
```
- Both threads are blocked waiting for each other to put into the queue, resulting in a deadlock. This is a common issue in systems that use bounded buffers or channels for communication.
- **Prevention**: Ensure that the order of operations is consistent and that there is always a path for at least one thread to make progress. Use unbounded queues or implement proper signaling between producers and consumers.

### Database connection pool deadlock
- A web application with a fixed-size database connection pool where all connections are in use, and threads are waiting for a connection to execute queries. If those threads are also holding locks on resources that other threads need to release connections, you can get a deadlock situation.

```
- Thread 1 grabs connection #1, waits for a free connection to do the second query.
- Thread 2 grabs connection #2, waits for a free connection to do the second query.
- Pool is exhausted; both wait forever for a slot the other refuses to release.
```
- **Prevention**: Use a larger connection pool, ensure that connections are released promptly, and avoid holding connections while waiting for other resources.

## LiveLock
- Threads are not blocked — they're actively running — but they keep changing state in response to each other without ever making actual progress.
- Usually from over-eager deadlock avoidance: a thread detects a potential conflict and "backs off" to let the other proceed, but both threads back off simultaneously, and this repeats indefinitely.
```java
// Simplified: two threads try to acquire two locks,
// and back off if they can't get the second one — but they always
// retry in lockstep, so neither wins.

boolean tryTransfer(Resource a, Resource b) {
    while (true) {
        if (a.lock.tryLock()) {
            try {
                if (b.lock.tryLock()) {
                    try {
                        return true; // success
                    } finally { b.lock.unlock(); }
                }
            } finally { a.lock.unlock(); }
        }
        // failed to get both — release and retry immediately
        // if both threads retry at the same rate, this loops forever
    }
}
```
- Each thread keeps grabbing and releasing locks, "doing work," but no transfer ever completes.
- **Detection**:
  - Harder to spot than deadlock because CPU usage stays high (threads are busy) but throughput/progress is zero. 
  - Look for: high CPU, no deadlock reported by thread dumps, but application-level progress (transactions completed, requests served) is stalled. 
  - Logging state transitions over time reveals repeating cycles.
- **Prevention**: 
  - Randomized backoff: instead of retrying immediately or at a fixed interval, wait a random amount of time before retrying  This breaks the "synchronized dance."
  - Priority/precedence rules: give one thread priority to proceed in a conflict rather than having both defer symmetrically. 
  - Limit retries: fail fast after N attempts rather than retrying forever.

## Starvation
- A thread is perpetually denied access to resources it needs to make progress, often because other threads are monopolizing those resources or Other threads keep "cutting in line".
- Common causes:
  - Unfair locks: a lock implementation that doesn't guarantee FIFO ordering may let some threads repeatedly "jump the queue."
  - Priority inversion/scheduling: low-priority threads never get CPU time because higher-priority threads always preempt them.
  - Greedy threads: a thread holding a shared resource for a long time (e.g., long-running synchronized blocks) starves others waiting on it.
- Detection:
  - Threads sit in BLOCKED or WAITING state for very long periods in thread dumps, but no cycle exists (ruling out deadlock). 
  - Monitoring wait-time metrics per-thread reveals large disparities — some threads always wait, others never do.
- Prevention strategies 
  - Fair locks: e.g., Java's new ReentrantLock(true) uses a fair ordering policy (FIFO), so threads acquire the lock in the order they requested it. 
  - Priority queues with aging: increase a waiting thread's priority the longer it waits, so it eventually gets serviced. 
  - Bounded lock hold times: don't hold locks longer than necessary; do expensive work outside synchronized blocks. 
  - Round-robin scheduling: ensures every thread gets a turn instead of a scheduler always favoring the same ones.


### Comparison at a glance
| Aspect                                | Deadlock                                      | Livelock                                            | Starvation                            |
|---------------------------------------|-----------------------------------------------|-----------------------------------------------------|---------------------------------------|
| **Thread state**                      | Blocked (waiting)                             | Active (busy, but unproductive)                     | Usually blocked/waiting               |
| **Root cause**                        | Circular resource dependency                  | Threads repeatedly react to each other and back off | Unfair scheduling/resource allocation |
| **CPU usage**                         | Low (threads idle)                            | High (threads busy "trying")                        | Varies                                |
| **Fix approach**                      | Lock ordering, timeouts, avoidance algorithms | Randomized backoff, priority rules                  | Fair locks, aging, bounded hold times |
| **Detectable via thread dump cycle?** | Yes                                           | No                                                  | No                                    |


