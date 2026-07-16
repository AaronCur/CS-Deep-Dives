[[Cheat Sheet](Cheat%20Sheet.md)
# Week 2 Deep Topic — Java Concurrency

**Suggested split:** ~45 min Threads/Executors + synchronized vs Lock, ~30 min volatile/deadlocks/CompletableFuture, then self-test.

---

## 1. Threads & Executors

### Creating threads
- Extend `Thread` or implement `Runnable` (prefer `Runnable` — decouples the task from the execution mechanism, and since Java has no multiple inheritance, implementing an interface leaves the class free to extend something else).
- `Callable<V>` — like `Runnable` but has a `call()` method that returns a value and can throw checked exceptions. Used with `ExecutorService.submit()`, which returns a `Future<V>`.

```java
Runnable r = () -> System.out.println("running");
Callable<Integer> c = () -> {
    Thread.sleep(100);
    return 42;
};
```

- A raw `new Thread(runnable).start()` works but gives you no pooling, no queuing, and no easy way to get a return value back — that's what executors solve.

### Why not raw threads in production code
- Each OS thread typically reserves ~1MB of stack space by default (`-Xss` controls this) — spinning up thousands is expensive in memory and in OS scheduling overhead.
- Thread creation/teardown itself has real cost; pooling amortizes it by reusing threads across many short tasks.
- Raw threads give you no lifecycle management, no queuing when work outpaces capacity, and no backpressure — a burst of requests each spawning a thread can take a service down.

### ExecutorService & thread pools
- `Executors.newFixedThreadPool(n)` — `n` threads, backed by an **unbounded** `LinkedBlockingQueue`. If tasks arrive faster than they're processed, the queue grows without limit → eventually OOM. The pool size stays fixed even under load, so there's no elasticity either.
- `Executors.newCachedThreadPool()` — no fixed limit on threads; creates new ones as needed, reuses idle ones, kills threads idle >60s. Under a sudden burst it can create an unbounded number of threads → resource exhaustion.
- `Executors.newSingleThreadExecutor()` — one thread, tasks run strictly in submission order. Useful when you need serialized access to something without hand-rolling synchronization.
- `Executors.newScheduledThreadPool(n)` — for delayed (`schedule`) or periodic (`scheduleAtFixedRate`, `scheduleWithFixedDelay`) tasks.
- **In practice:** senior-level answer is to construct `ThreadPoolExecutor` directly so every knob is explicit and bounded, rather than relying on `Executors` factories that hide an unbounded queue or unbounded thread count. This is a common "gotcha" interview question — if asked "what's wrong with `Executors.newFixedThreadPool`", the unbounded queue is the answer.

### ThreadPoolExecutor core knobs
```java
new ThreadPoolExecutor(
    corePoolSize,          // threads kept alive even when idle
    maximumPoolSize,       // ceiling on total threads
    keepAliveTime, unit,   // how long idle threads beyond core survive
    new ArrayBlockingQueue<>(capacity),  // bounded queue — the important part
    new ThreadPoolExecutor.CallerRunsPolicy()  // what to do when full
);
```
- **Task admission order:** if running threads < core → spin up a new thread for the task, even if other core threads are idle. Once core is full, new tasks go into the queue. Once the queue is full too, the pool creates threads above core up to `maximumPoolSize`. Once that's also maxed and the queue is full → the `RejectedExecutionHandler` runs.
- Common rejection policies: `AbortPolicy` (throws `RejectedExecutionException`, default), `CallerRunsPolicy` (runs the task on the calling thread — natural backpressure, since the caller is now busy and can't submit more work as fast), `DiscardPolicy`, `DiscardOldestPolicy`.
- Choosing a **bounded** queue capacity plus a real `maximumPoolSize` is what actually protects you from the unbounded-growth failure modes above.

### Shutdown
- `shutdown()` — stops accepting new tasks; already-submitted (queued or running) tasks run to completion.
- `shutdownNow()` — attempts to stop everything immediately: doesn't start queued tasks, and interrupts running ones (interruption is cooperative — a task ignoring `InterruptedException`/`isInterrupted()` won't actually stop).
- Always pair a shutdown call with `awaitTermination(timeout, unit)` and handle the case where it times out — otherwise you can leak threads (e.g. a non-daemon thread pool keeping the JVM alive on app shutdown), which is a real production bug pattern worth mentioning if asked about it.

---

## 2. synchronized vs Lock

### synchronized
- Intrinsic/monitor lock built into every Java object (every object has an associated monitor, whether you use it or not).
- Method form locks on `this` (instance methods) or the `Class` object (static methods). Block form lets you lock on an explicit object and scope the critical section tightly:

```java
public class Counter {
    private int count = 0;
    private final Object lock = new Object();

    public void increment() {
        synchronized (lock) {   // block form — narrower scope than synchronizing the whole method
            count++;
        }
    }
}
```
- **Reentrant:** if a thread already holds the lock on an object, it can enter another `synchronized` block on the same object without deadlocking itself (e.g. one synchronized method calling another synchronized method on the same instance).
- Released automatically on block/method exit — including when an exception propagates out — so it can never be "leaked" by a forgotten unlock call the way a manual `Lock` can.
- Limitations that motivate `Lock`: can't attempt the lock and give up (`tryLock`), can't time out, a thread blocked waiting on it can't be interrupted, no fairness guarantee (JVM can let a thread barge ahead of ones that have been waiting longer), and you only get one implicit condition (`wait`/`notify`/`notifyAll` on that same object).

### Lock (java.util.concurrent.locks)
- `ReentrantLock` is the standard implementation — same reentrancy guarantee as `synchronized`, but as an explicit object you interact with via method calls.
- **Must** manually unlock, and it has to be in `finally` or a lock leaks on exception, permanently blocking every other thread that needs it:

```java
private final ReentrantLock lock = new ReentrantLock();

public void increment() {
    lock.lock();
    try {
        count++;
    } finally {
        lock.unlock();   // non-negotiable — if this is missed, every other caller blocks forever
    }
}
```
- Extra capabilities `synchronized` doesn't have:
  - `tryLock()` — returns immediately with true/false instead of blocking; `tryLock(timeout, unit)` — waits up to a bound, then gives up. Useful to avoid a thread stuck waiting indefinitely, and as a building block for deadlock avoidance (back off and retry rather than block forever).
  - `lockInterruptibly()` — a thread blocked waiting for this lock can be interrupted out of the wait (e.g. to support task cancellation), where a thread blocked on `synchronized` cannot be.
  - Fairness option (`new ReentrantLock(true)`) — roughly FIFO acquisition order instead of allowing barging, at some throughput cost since it prevents an idle-and-ready thread from jumping the queue.
  - Can back multiple `Condition` objects via `newCondition()` — e.g. a bounded buffer can have a separate "not full" and "not empty" condition instead of one shared `wait`/`notifyAll` set, which reduces unnecessary wake-ups.
- `ReadWriteLock` / `ReentrantReadWriteLock` — `readLock()` can be held by multiple threads concurrently as long as no thread holds `writeLock()`; a writer gets fully exclusive access. Good fit for state that's read far more often than it's written (e.g. a config cache refreshed occasionally but read constantly).

### When to reach for which
- Default to `synchronized` for straightforward mutual exclusion — it's less error-prone because release is automatic, and it's what most interviewers expect as the baseline answer.
- Reach for `Lock` specifically when you need tryLock/timeouts, interruptible waits, fairness, multiple wait-conditions, or read/write separation — be ready to name *which* of these you need, since "Lock is more flexible" alone is a weak interview answer.

---

## 3. volatile

- Guarantees **visibility**, not atomicity. Without `volatile`, a thread may cache a field's value in a CPU register or core-local cache and never see another thread's update; marking it `volatile` forces every read to come from main memory and every write to flush there immediately.
- Also prevents certain instruction reordering: a volatile read/write establishes a **happens-before** edge — everything a thread wrote before the volatile write is guaranteed visible to another thread that performs the volatile read afterward. This matters beyond the field itself (it's part of how safe publication of an object works).
- **Does not** make compound actions atomic:
```java
private volatile int count = 0;
public void increment() {
    count++;   // still a read-modify-write race: two threads can read the same
               // value before either writes back, and one increment gets lost
}
```
  For atomic compound operations, use `synchronized`, `Lock`, or the `java.util.concurrent.atomic` classes.
- Classic *correct* use case — a stop flag, where the operation is a single write/read, not a compound update:
```java
private volatile boolean running = true;
public void stop() { running = false; }               // writer thread
public void run() { while (running) { /* work */ } }   // worker thread sees the flip promptly
```
- `AtomicInteger`/`AtomicLong`/`AtomicReference` etc. use a volatile field internally plus CAS (compare-and-swap, a hardware atomic instruction) to implement lock-free atomic updates like `incrementAndGet()` — the right fix for the `count++` race above.

---

## 4. Deadlocks

### The four necessary conditions (all must hold simultaneously)
1. **Mutual exclusion** — a resource can only be held by one thread at a time.
2. **Hold and wait** — a thread holds at least one resource while waiting to acquire another.
3. **No preemption** — a resource can't be forcibly taken from the thread holding it; it must be released voluntarily.
4. **Circular wait** — a cycle exists: thread 1 waits on a resource held by thread 2, which waits on one held by thread 3, ... back to thread 1.

Break any one of the four and deadlock becomes impossible — that's the basis for every prevention strategy below.

### Classic example
```java
// Thread A
synchronized (resource1) {
    synchronized (resource2) { ... }
}
// Thread B — locks acquired in the OPPOSITE order
synchronized (resource2) {
    synchronized (resource1) { ... }
}
```
If A grabs `resource1` and B grabs `resource2` at nearly the same time, A then blocks waiting for `resource2` (held by B) while B blocks waiting for `resource1` (held by A). Neither can ever proceed — classic circular wait.

### Prevention/mitigation strategies
- **Lock ordering** — establish one global order (e.g. by object hash code or an assigned ID) and always acquire locks in that order everywhere in the codebase. This directly breaks circular wait — in the example above, if both threads always locked `resource1` before `resource2`, deadlock couldn't occur. Most common real-world fix.
- **tryLock with timeout** — instead of blocking indefinitely, a thread that can't get the second lock within a bound gives up, releases what it holds, and retries later. Breaks the practical effect of "no preemption."
- Keep lock scope and hold time as small as possible, and avoid calling into code you don't control (e.g. a callback) while holding a lock — that code might try to acquire a lock of its own.
- Prefer higher-level, already-deadlock-safe concurrency utilities (concurrent collections, `CompletableFuture` pipelines, executors) over hand-rolled nested locking wherever the design allows it.

### Detecting deadlocks
- A thread dump (`jstack <pid>`, or the thread dump feature in a profiler/APM tool) explicitly reports **"Found one Java-level deadlock"** along with the exact cycle of threads and the locks each is waiting on/holding — usually the fastest way to confirm and diagnose one in production.

---

## 5. CompletableFuture

- `Future` (from `ExecutorService.submit()`) is blocking-only: `get()` blocks the calling thread until the task finishes, with no way to chain a follow-up action or combine it with another `Future` without more blocking. `CompletableFuture<T>` (Java 8+) fixes this — it supports non-blocking async pipelines.

```java
CompletableFuture<String> pipeline = CompletableFuture
    .supplyAsync(() -> fetchUserId())          // async, returns a value
    .thenApplyAsync(id -> fetchUserName(id))   // next async stage, depends on prior result
    .thenApply(name -> "Hello, " + name)       // cheap sync transform
    .exceptionally(ex -> "fallback: " + ex.getMessage());
```

- Key methods:
  - `supplyAsync(Supplier<T>)` — start async work that returns a value.
  - `runAsync(Runnable)` — start async work with no return value.
  - `.thenApply(fn)` — transform the result once available. May run on the thread that completed the previous stage (possibly even the calling thread, if the future was already done) — cheap, but keep the function fast/non-blocking.
  - `.thenApplyAsync(fn)` — same transform, but explicitly submitted to the common `ForkJoinPool` (or a supplied `Executor`) rather than possibly running inline.
  - `.thenCompose(fn)` — use when the next step **itself returns** a `CompletableFuture<T>` (e.g. it makes another async call) — flattens `CompletableFuture<CompletableFuture<T>>` into `CompletableFuture<T>`, like `flatMap` for futures. Using `thenApply` here would give a nested future instead.
  - `.thenCombine(other, fn)` — combine the results of two independent futures once both complete (e.g. two parallel API calls).
  - `.exceptionally(fn)` — supply a fallback value if the pipeline threw. `.handle(fn)` — runs regardless of success/failure, receives `(result, exception)` and can inspect both.
  - `CompletableFuture.allOf(f1, f2, f3)` — completes when all inputs complete (returns `Void`, so results are pulled from each future individually with `.join()` afterward). `.anyOf(...)` — completes as soon as any one input completes.
- **Default executor:** stages without an explicit executor run on the shared common `ForkJoinPool`, sized around available CPU cores and meant for short CPU-bound work. If a stage does blocking I/O (a DB call, an HTTP request), it can occupy a pool thread for a long time and starve unrelated work elsewhere in the JVM that also relies on the common pool (e.g. parallel streams). Pass a dedicated `Executor` to the `...Async` variants for I/O-bound stages.
- `thenApply` vs `thenApplyAsync` is a frequent interview trip-up — be ready to state precisely: `thenApply` *may* run inline on whichever thread completes the prior stage, `thenApplyAsync` *guarantees* a hand-off to the pool (default or supplied).

---

## Self-Test Questions

1. Why do the `Executors` factory methods (e.g. `newFixedThreadPool`) get flagged in production code reviews, and what would you use instead?
2. Explain the difference between `synchronized` and `ReentrantLock` in terms of what happens if an exception is thrown while the lock is held.
3. Is `volatile` enough to make a counter increment (`count++`) thread-safe? Why or why not — what would you use instead?
4. Name the four conditions required for a deadlock, and describe the most common real-world technique to prevent it.
5. What's the practical difference between `thenApply` and `thenApplyAsync` on a `CompletableFuture`, and why does it matter for I/O-bound work?
6. When would you reach for `ReadWriteLock` instead of a plain `ReentrantLock`?
7. Walk through what happens inside a `ThreadPoolExecutor` when a new task arrives and the core pool is full but the queue has capacity.
