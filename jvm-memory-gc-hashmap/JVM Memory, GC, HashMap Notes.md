[Cheat Sheet](Cheat%20Sheet.md)
# Week 1 Deep Topic: JVM Memory Model, GC Basics, HashMap Internals

---

## 1. JVM Memory Model

### 1.1 Runtime Data Areas — the full picture

| Area | Shared or per-thread | Stores | Notes |
|---|---|---|---|
| Heap | Shared across all threads | All objects, arrays, instance fields | Managed by GC |
| Stack (JVM stack) | Per-thread | Stack frames: local vars, operand stack, return address | Primitives stored by value, objects stored as references |
| Metaspace (Method Area) | Shared | Class metadata, method bytecode, static variables, runtime constant pool | Native memory since Java 8 (replaced PermGen) |
| PC Register | Per-thread | Address of the currently executing bytecode instruction | Undefined for native methods |
| Native Method Stack | Per-thread | Support for native (JNI) method calls | Rarely asked about, good to mention exists |

**Interview framing:** "Thread-shared" areas (Heap, Metaspace) exist once per JVM. "Thread-private" areas (Stack, PC Register, Native Method Stack) exist once per thread.

### 1.2 Heap in detail

- Single heap per JVM, divided (in generational collectors) into:
  - **Young Generation**: Eden + two Survivor spaces (S0/S1). Most objects are created here and die young ("weak generational hypothesis").
  - **Old/Tenured Generation**: objects that survive enough young GC cycles get *promoted* here.
- Object lifecycle: allocated in Eden → survives a Minor GC → moved to a Survivor space → after surviving N GC cycles (tenuring threshold) → promoted to Old Gen.
- Heap size controlled by `-Xms` (initial) and `-Xmx` (max).

```java
public class MemoryExample {
    public static void main(String[] args) {
        int localPrimitive = 42;              // value lives on the stack frame for main()
        MemoryExample obj = new MemoryExample(); // reference "obj" lives on the stack,
                                                   // the actual object lives on the heap
        obj.doWork(localPrimitive);
    }

    void doWork(int n) {                      // new stack frame pushed here
        int[] arr = new int[n];               // array object → heap; "arr" reference → stack
    }                                          // frame popped when doWork() returns
}
```

### 1.3 Stack in detail

- Each method call pushes a **stack frame** containing: local variables, operand stack, reference to runtime constant pool, return address.
- Frame is popped when the method returns.
- Thread-safe by construction — no other thread can see your stack.
- `-Xss` controls per-thread stack size (default ~512KB–1MB depending on OS/arch).
- `StackOverflowError` — usually uncontrolled/deep recursion:

```java
public int badRecursion(int n) {
    return badRecursion(n + 1);   // no base case → keeps pushing frames until
}                                  // the stack is exhausted → StackOverflowError
```

### 1.4 Metaspace

- Replaced PermGen in Java 8 specifically to remove the fixed-size `OutOfMemoryError: PermGen space` problem.
- Lives in **native memory**, not the heap — grows dynamically, bounded by `-XX:MaxMetaspaceSize` (unbounded by default, bounded by available system memory).
- Stores: class structure, method bytecode, field/method metadata, constant pool.
- Common failure mode: **classloader leaks** — classes can't be unloaded until their defining ClassLoader becomes garbage-collectible (common in app servers doing hot redeploys, or heavy use of dynamic proxies/bytecode generation).

### 1.5 Common OutOfMemoryErrors — know what each means

- `OutOfMemoryError: Java heap space` → heap exhausted (leak or genuinely too much live data)
- `OutOfMemoryError: Metaspace` → too many/leaking classes
- `StackOverflowError` → stack exhausted (not an OOME — separate error type)
- `OutOfMemoryError: GC overhead limit exceeded` → GC running constantly but reclaiming almost nothing
- `OutOfMemoryError: Unable to create new native threads` → OS thread limit hit, or too many threads each reserving stack space

---

## 2. Garbage Collection Basics

### 2.1 Core idea

- Java uses **automatic memory management**: the GC identifies objects with no live references (unreachable from GC roots — stack locals, static fields, active threads, JNI refs) and reclaims their memory.
- `System.gc()` is only a *hint* — JVM is free to ignore it.

### 2.2 Generational hypothesis

Most objects die young → so it's cheaper to collect Young Gen frequently (fast, small) and Old Gen rarely (slow, large).

- **Minor GC**: collects Young Gen only. Frequent, usually fast (stop-the-world but short).
- **Major/Full GC**: collects Old Gen (often the whole heap). Less frequent, much more expensive — this is what causes noticeable app pauses.

### 2.3 GC algorithm phases (conceptually)

1. **Mark** — traverse from GC roots, mark all reachable objects as live.
2. **Sweep** — reclaim memory of unmarked (dead) objects.
3. **Compact** — move surviving objects together to eliminate fragmentation (not all collectors do this every cycle).

### 2.4 Collectors you should be able to compare

| Collector | Design goal | Notes |
|---|---|---|
| Serial GC | Simplicity, single-threaded | Good for small heaps / single-core, not for servers |
| Parallel GC | Throughput | Multiple threads for GC work, still stop-the-world for major collections |
| CMS (Concurrent Mark Sweep) | Low pause time | **Deprecated in Java 9, removed in Java 14** — don't lead with this unless asked about history |
| **G1 (Garbage First)** | Balanced throughput/latency, predictable pauses | Default collector since Java 9. Splits heap into regions, collects the regions with the most garbage first |
| **ZGC** | Ultra-low pause (sub-ms), scales to huge heaps | Fully concurrent, pause times don't scale with heap size |
| Shenandoah | Similar goals to ZGC | Concurrent compaction, low pause |

**Likely interview question:** "Which collector would you pick and why?" → Default answer: G1 for general-purpose server apps (good balance); ZGC/Shenandoah if you have very large heaps and need very low pause times (e.g., latency-sensitive trading/real-time systems).

### 2.5 Practical tuning knobs (know these exist, not memorize every flag)

- `-Xms` / `-Xmx` — initial/max heap size
- `-XX:+UseG1GC`, `-XX:+UseZGC` — collector selection
- `-XX:MaxMetaspaceSize` — cap metaspace growth
- `-Xss` — per-thread stack size

```bash
# Example: server app, G1 collector, 4GB heap, capped metaspace
java -Xms2g -Xmx4g -XX:+UseG1GC -XX:MaxMetaspaceSize=256m -jar app.jar

# Example: latency-sensitive app on a large heap
java -Xmx32g -XX:+UseZGC -jar app.jar
```

---

## 3. HashMap Internals (Java 8+)

### 3.1 Core structure

- Backed by an array of buckets: `Node<K,V>[] table`.
- Each bucket holds either:
  - a **linked list** of entries that hash to the same bucket (the common case), or
  - a **red-black tree** if that bucket gets too crowded (Java 8+ optimization).

### 3.2 Put/get flow

1. Compute `hashCode()` of the key.
2. Apply HashMap's internal **hash spreading function** (XORs the hash with its own upper bits) to reduce clustering from poor hash functions.
3. Compute bucket index via `(table.length - 1) & hash` — this is why capacity is always a power of two (turns modulo into a fast bitwise AND).
4. Walk the bucket: compare hash first (cheap), then `equals()` (authoritative) to find/confirm the key.

**Interview line to have ready:** *"hashCode narrows down which bucket to search; equals confirms actual identity. HashMap only calls equals when hashes already match."*

```java
// Simplified version of what HashMap does internally (JDK source, paraphrased)
static int spread(int h) {
    return h ^ (h >>> 16);          // mix high bits into low bits
}

int bucketIndex = (table.length - 1) & spread(key.hashCode());
```

### 3.3 Resizing

- Default initial capacity: 16. Default load factor: 0.75.
- `threshold = capacity × loadFactor` (e.g., 16 × 0.75 = 12).
- When `size > threshold`, the table **doubles** in capacity and all entries are rehashed into the new table.
- Trade-off: lower load factor = fewer collisions but more memory and more frequent resizes; higher load factor = less memory but more collisions.
- **Practical tip to mention in interviews:** if you know the expected size up front, pre-size the HashMap (`new HashMap<>(expectedSize / 0.75f)`) to avoid repeated resize/rehash costs.

```java
// Bad: default capacity (16), will resize multiple times while loading 10,000 entries
Map<String, Object> cache = new HashMap<>();

// Better: pre-sized to avoid repeated doubling + rehashing
Map<String, Object> cache = new HashMap<>((int) (10_000 / 0.75f) + 1);
```

### 3.4 Treeification (Java 8+)

- If a single bucket's linked list grows to **8** entries (`TREEIFY_THRESHOLD`) **and** the table capacity is at least **64** (`MIN_TREEIFY_CAPACITY`), that bucket converts to a red-black tree → worst case O(n) becomes O(log n).
- If the table is smaller than 64, HashMap prefers to **resize instead of treeify** — spreading entries out is cheaper than maintaining a tree on a small table.
- If a tree bucket shrinks back down to **6** entries (`UNTREEIFY_THRESHOLD`), it reverts to a linked list.
- Why this exists: protects against worst-case O(n) chains — either from bad hash functions or adversarially crafted keys (a real historical DoS vector against naive hash tables).

### 3.5 equals/hashCode contract — classic interview trap

- If `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` **must** be true.
- The reverse is not required (different objects can share a hash — that's a collision, not a bug).
- Violating this contract (e.g., overriding `equals` but not `hashCode`) causes HashMap to "lose" entries — you can `put` something and then `get` returns null because it's looking in the wrong bucket.
- **Mutable keys are dangerous**: if a key's hashCode changes after insertion (e.g., mutating a field used in hashCode), the entry becomes unreachable — it's still in the table, just in the wrong bucket relative to its current hash.

```java
class BadKey {
    int id;
    BadKey(int id) { this.id = id; }

    @Override
    public boolean equals(Object o) {
        return o instanceof BadKey && ((BadKey) o).id == this.id;
    }
    // no hashCode() override → uses Object's identity hashCode
    // → equal keys can land in different buckets → map.get() silently returns null
}

// Mutable-key bug:
Map<BadKey, String> map = new HashMap<>();
BadKey key = new BadKey(1);
map.put(key, "value");
key.id = 2;                 // mutate the field used in equals/hashCode after insertion
map.get(key);                // often returns null — key now hashes to a different bucket
```

### 3.6 Other things worth knowing cold

- HashMap allows **one null key** (goes to bucket 0, since `hash(null) = 0`) and multiple null values.
- **Not thread-safe.** Concurrent modification during resize in pre-Java-8 versions could corrupt the linked list into a cycle, causing an infinite loop on `get()` (classic production incident story — good behavioral/technical crossover anecdote). Java 8 fixed the resize algorithm to avoid this, but HashMap is still not safe for concurrent use — use `ConcurrentHashMap` instead.
- Average time complexity: O(1) for get/put/remove; worst case O(log n) post-treeification (was O(n) pre-Java-8).

---

## 4. Suggested Resources

**JVM memory model**
- [Java Memory Management Explained — DigitalOcean](https://www.digitalocean.com/community/tutorials/java-jvm-memory-model-memory-management-in-java) — probably the most complete single article: covers heap/stack/metaspace/PC register, GC phases, and collector comparison (G1/ZGC/Shenandoah) in one place.
- [A Deep Dive into the JVM Memory Model — HeapHero](https://blog.heaphero.io/a-deep-dive-into-the-jvm-memory-model-how-heap-stack-and-metaspace-function-and-fail/) — good for the failure-mode angle (which OOME maps to which region), useful for "how would you debug X" style questions.
- Oracle's official JVM Spec, Chapter 2.5 (Runtime Data Areas) — the primary source, worth skimming once for precision on terminology.

**Garbage Collection**
- Oracle's "Garbage Collection Tuning Guide" (search current version for your JDK) — the canonical reference for G1 internals and tuning flags.
- Any recent writeup comparing G1 vs ZGC vs Shenandoah — search for something current since collector defaults/behavior have shifted across JDK versions (you're likely on JDK 17/21 in interviews — check what's default there).

**HashMap internals**
- [HashMap Internals in Java — Janak Avhad, Medium](https://medium.com/@avhadjanak/hashmap-internals-in-java-636206c36c9b) — strong end-to-end walkthrough: buckets, treeification thresholds, the Java 7 resize/infinite-loop bug story, null key handling. Good one to read fully.
- [Deep Dive into Java HashMap: Performance Optimizations and Pitfalls — Java Code Geeks](https://www.javacodegeeks.com/2025/09/deep-dive-into-java-hashmap-performance-optimizations-and-pitfalls.html) — good for the "pitfalls" framing (mutable keys, thread-safety, memory footprint) which maps well to behavioral/practical interview questions.
- Actually reading the JDK source for `HashMap.java` (`resize()`, `putVal()`, `treeifyBin()`) once is genuinely worth 20 minutes — most "how does X work internally" answers get much more confident once you've seen the real code, not just descriptions of it.

---

## 5. Self-Test Questions (close the notes and answer these)

1. Why did Java 8 replace PermGen with Metaspace? What specific problem did this solve?
2. Walk through what happens, step by step, when a Minor GC runs.
3. Why is HashMap capacity always a power of two?
4. What are the two conditions that must both be true before a bucket treeifies?
5. Explain why overriding `equals()` without overriding `hashCode()` breaks HashMap.
6. What's the difference between G1 and ZGC, and when would you choose one over the other?
7. Why is a mutable object a bad HashMap key?
