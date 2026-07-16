# Concurrency Cheat Sheet (Week 2 Deliverable)

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## Threads & Executors
| Concept | Key point |
|---|---|
| `Runnable` vs `Callable` | Callable returns a value + can throw checked exceptions |
| `Executors.newFixedThreadPool` | Unbounded queue → OOM risk under sustained load |
| `Executors.newCachedThreadPool` | Unbounded threads → resource exhaustion risk |
| `ThreadPoolExecutor` (direct) | Preferred in prod: control core/max size, queue, rejection policy |
| Task admission order | core threads → queue → max threads → rejection handler |
| Shutdown | `shutdown()` = graceful; `shutdownNow()` = force; always `awaitTermination()` |

## synchronized vs Lock
| | synchronized | ReentrantLock |
|---|---|---|
| Release on exception | Automatic | Manual — must unlock in `finally` |
| tryLock / timeout | No | Yes |
| Interruptible wait | No | `lockInterruptibly()` |
| Fairness option | No | Yes (`new ReentrantLock(true)`) |
| Multiple wait-sets | No (one implicit monitor) | Yes, via `newCondition()` |
| Default choice | Simple mutual exclusion | Need tryLock/fairness/interruptibility |

`ReadWriteLock` → many concurrent readers, exclusive writer. Use for read-heavy shared state.

## volatile
- Guarantees **visibility** + ordering (happens-before), **not atomicity**.
- `count++` on a volatile field is still a race → use `AtomicInteger`/`synchronized`/`Lock`.
- Good use: a `boolean running` stop-flag read by another thread.

## Deadlocks
**4 conditions (all required):** mutual exclusion, hold-and-wait, no preemption, circular wait.

**Prevent:** consistent global lock ordering (breaks circular wait) > tryLock+timeout+backoff > minimize hold time / avoid calling unknown code while locked.

**Detect:** thread dump (`jstack`) → "Found one Java-level deadlock."

## CompletableFuture
| Method | Use |
|---|---|
| `supplyAsync` / `runAsync` | Start async work (with/without return value) |
| `thenApply` | Transform result — may run on completing thread |
| `thenApplyAsync` | Transform result — hands off to pool (common ForkJoinPool by default) |
| `thenCompose` | Flatten nested futures (flatMap) — next step returns a CF |
| `thenCombine` | Combine two independent futures |
| `exceptionally` / `handle` | Error handling |
| `allOf` / `anyOf` | Wait on multiple futures |

⚠️ Default pool = common `ForkJoinPool` — pass a dedicated executor for blocking I/O work to avoid starving other tasks.
