# Java Concurrency: `synchronized` vs `volatile` vs `AtomicInteger`

These are three different mechanisms in Java for handling concurrent access to shared data.

---

## `synchronized`

Provides **mutual exclusion** (locking). Only one thread can execute a synchronized block/method at a time.

```java
private int count = 0;

public synchronized void increment() {
    count++;  // read-modify-write is atomic under the lock
}
```

- **Guarantees**: atomicity + visibility (memory barrier on entry/exit)
- **Cost**: acquiring/releasing a monitor lock; threads may block
- **Use when**: you need compound operations to be atomic (e.g., check-then-act, read-modify-write)

---

## `volatile`

Ensures **visibility** only — reads/writes go directly to main memory, not a CPU cache.

```java
private volatile boolean running = true;

// Thread A
public void stop() { running = false; }

// Thread B
while (running) { /* ... */ }
```

- **Guarantees**: visibility (every read sees the latest write)
- **Does NOT guarantee**: atomicity of compound operations like `count++` (that's three operations: read, add, write)
- **Cost**: very low — no locking
- **Use when**: one thread writes, others only read; or for simple flags

---

## `AtomicInteger`

Provides **lock-free atomic operations** via CPU-level CAS (Compare-And-Swap) instructions.

```java
private AtomicInteger count = new AtomicInteger(0);

public void increment() {
    count.incrementAndGet();  // truly atomic, no lock needed
}
```

- **Guarantees**: atomicity + visibility, without locking
- **Key methods**: `get()`, `set()`, `incrementAndGet()`, `compareAndSet()`, etc.
- **Cost**: slightly higher than `volatile` but much lower than `synchronized`
- **Use when**: you need atomic read-modify-write on a single integer (counters, sequence generators)

---

## Quick Comparison

| Feature | `synchronized` | `volatile` | `AtomicInteger` |
|---|---|---|---|
| Mutual exclusion | Yes | No | No (uses CAS) |
| Visibility guarantee | Yes | Yes | Yes |
| Atomic compound ops | Yes | No | Yes |
| Blocks threads | Yes | No | No |
| Performance | Lowest | Highest | Middle |
| Scope | Any shared state | Single variable | Single `int` |

---

## Rule of Thumb

- Need a **flag** shared across threads? → `volatile`
- Need an **atomic counter**? → `AtomicInteger`
- Need to protect a **block of code** or **multiple variables together**? → `synchronized`
