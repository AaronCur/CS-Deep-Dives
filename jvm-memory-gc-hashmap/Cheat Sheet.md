# JVM Memory / GC / HashMap Cheat Sheet (Week 1 Deliverable)

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## Runtime Data Areas
| Area | Shared/per-thread | Stores |
|---|---|---|
| Heap | Shared | Objects, arrays |
| Stack | Per-thread | Local vars, stack frames, return addresses |
| Metaspace | Shared | Class metadata, static vars, constant pool (native memory since Java 8) |
| PC Register | Per-thread | Address of next bytecode instruction |
| Native Method Stack | Per-thread | JNI/native call support |

- Heap → `-Xms` / `-Xmx`. Stack → `-Xss`. Metaspace → `-XX:MaxMetaspaceSize`.
- Young Gen (Eden + 2 Survivor spaces) → most objects die here. Old Gen → promoted survivors.
- `StackOverflowError` = stack exhausted (not an OOME). `OutOfMemoryError: Metaspace` = classloader leak / too many classes.

## Garbage Collection
- **Minor GC** = Young Gen only, frequent, fast. **Major/Full GC** = Old Gen (often whole heap), expensive, causes visible pauses.
- Phases: **Mark** (find live objects from GC roots) → **Sweep** (reclaim dead) → **Compact** (defragment).
- `System.gc()` is a hint only — JVM can ignore it.

| Collector | Goal | Note |
|---|---|---|
| Serial | Simplicity | Single-threaded, small heaps only |
| Parallel | Throughput | Multi-threaded, still stop-the-world |
| CMS | Low pause | ⚠️ Removed in Java 14 |
| **G1** | Balanced | Default since Java 9 — go-to general answer |
| **ZGC** | Sub-ms pause, huge heaps | Fully concurrent |
| Shenandoah | Similar to ZGC | Concurrent compaction |

**"Which collector?"** → G1 by default; ZGC/Shenandoah for very large heaps + strict low-latency needs.

## HashMap Internals
- Structure: `Node<K,V>[] table` → each bucket = linked list, or red-black tree if crowded.
- Index: `(table.length - 1) & spread(hashCode())` → capacity is always a power of 2.
- **hashCode narrows the bucket; equals confirms identity.** equals only called on hash match.

| Constant | Value | Meaning |
|---|---|---|
| Default capacity | 16 | Initial bucket array size |
| Default load factor | 0.75 | Resize threshold = capacity × loadFactor |
| `TREEIFY_THRESHOLD` | 8 | Bucket size that triggers treeify (linked list → red-black tree) |
| `MIN_TREEIFY_CAPACITY` | 64 | Table must be ≥ this size to treeify; otherwise resize instead |
| `UNTREEIFY_THRESHOLD` | 6 | Tree shrinks back to linked list below this size |

- Resize = table **doubles**, all entries rehashed. Pre-size if expected count is known: `new HashMap<>((int)(n/0.75f)+1)`.
- **equals/hashCode contract:** `a.equals(b) == true` → `a.hashCode() == b.hashCode()` must hold. Reverse not required.
- Mutable keys = bug risk: mutating a field used in `hashCode()` after insertion strands the entry in the wrong bucket.
- One `null` key allowed (bucket 0). Multiple `null` values allowed.
- **Not thread-safe** — pre-Java-8 concurrent resize could cycle the linked list → infinite loop on `get()`. Use `ConcurrentHashMap` for concurrent access.
- Avg complexity: O(1); worst case O(log n) post-treeification (was O(n) pre-Java-8).

⚠️ Common trip-ups: overriding `equals()` without `hashCode()`; using mutable objects as keys; assuming HashMap is thread-safe.
