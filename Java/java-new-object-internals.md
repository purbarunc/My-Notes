# Java Interview: What Happens Internally When You Create an Object Using `new`?

> **Target Level:** 10 Years Experienced Java Developer

---

## The Statement

```java
MyClass obj = new MyClass();
```

---

## 1. Bytecode Compilation

The compiler generates **two** bytecode instructions:

- `new` — allocates memory
- `invokespecial` — calls the constructor `<init>`

---

## 2. Class Loading (if not already loaded)

Before allocation, the JVM checks if `MyClass` is already loaded. If not:

- **Loading**: `.class` file is read by the ClassLoader (Bootstrap → Extension/Platform → Application)
- **Linking**:
  - **Verification**: Bytecode is validated for correctness and security
  - **Preparation**: Static fields are allocated and set to default values (`0`, `null`, `false`)
  - **Resolution**: Symbolic references (e.g., class names) are resolved to direct memory references
- **Initialization**: Static initializer blocks and static field assignments run (`<clinit>`)

---

## 3. Memory Allocation in the Heap

The JVM allocates memory in the **Eden Space** (Young Generation of the Heap).

Two common allocation strategies:

- **Bump-the-pointer**: If heap is contiguous, simply moves a pointer forward — O(1), very fast
- **TLAB (Thread-Local Allocation Buffer)**: Each thread has its own private buffer in Eden — avoids synchronization overhead for concurrent allocations

Size determined by: instance fields (including inherited) + object header overhead.

---

## 4. Object Header Initialization

Every Java object has a **header** (typically 12–16 bytes):

| Part             | Size              | Purpose                                              |
|------------------|-------------------|------------------------------------------------------|
| **Mark Word**    | 8 bytes           | Hashcode, GC age, lock state (biased/thin/fat lock)  |
| **Klass Pointer**| 4 bytes (compressed) | Pointer to Class metadata in Metaspace            |
| **Array Length** | 4 bytes           | Only for arrays                                      |

---

## 5. Zero-Initialization of Fields

All instance fields are **zero-initialized** by the JVM:

- `int` → `0`, `boolean` → `false`, references → `null`

This happens **before** the constructor runs — guaranteeing the Java memory model's safety guarantee.

---

## 6. Constructor Execution (`invokespecial <init>`)

The constructor call chain executes:

1. `super()` is called first (implicitly or explicitly) — bubbles up to `Object.<init>()`
2. Instance initializer blocks run
3. Instance field assignments run
4. Rest of constructor body executes

This ensures the object is fully initialized top-down.

---

## 7. Reference Assignment

The reference `obj` on the stack now points to the newly created object in the heap. The object is live and eligible for GC tracking.

---

## Summary Flow

```
new keyword
    └─► Class loaded? ──No──► ClassLoader → Link → Initialize
              │
             Yes
              ▼
        Allocate in TLAB/Eden (bump-pointer)
              ▼
        Write Object Header (Mark Word + Klass Pointer)
              ▼
        Zero-initialize all instance fields
              ▼
        invokespecial <init> (constructor chain: Object → Parent → Child)
              ▼
        Return reference to caller
```

---

---

# Follow-up Depth Points for 10 YOE Java Developer

---

## 1. Escape Analysis & Stack Allocation (JIT Optimization)

### What is Escape Analysis?

The JIT compiler (C2 compiler) analyzes whether an object **"escapes"** the scope where it was created. An object "escapes" if:

- It is returned from the method
- It is passed to another method
- It is assigned to a field of another object
- It is accessed by another thread

### If the object does NOT escape:

The JVM can apply three optimizations:

#### a) Stack Allocation

```java
void process() {
    Point p = new Point(3, 4); // Does not escape this method
    double dist = Math.sqrt(p.x * p.x + p.y * p.y);
    // p is never returned, stored, or passed elsewhere
}
```

- Object is allocated on the **thread stack** instead of the heap
- Automatically reclaimed when method returns — **zero GC pressure**
- Stack allocation is extremely fast (single pointer decrement)

#### b) Scalar Replacement

Goes one step further — the object is **never created at all**:

```java
// JIT rewrites this internally as if:
void process() {
    int p_x = 3;   // Fields promoted to local variables
    int p_y = 4;
    double dist = Math.sqrt(p_x * p_x + p_y * p_y);
}
```

- Object's fields become CPU registers or stack slots
- No allocation whatsoever

#### c) Synchronization Elision

If a lock never escapes, `synchronized` on it is **completely removed**:

```java
void process() {
    Object lock = new Object();
    synchronized(lock) { // Lock is local — JIT removes this entirely
        // ...
    }
}
```

### How to verify Escape Analysis is working:

```bash
# Enable EA (on by default in modern JVMs)
-XX:+DoEscapeAnalysis

# Print EA decisions
-XX:+PrintEscapeAnalysis

# Confirm scalar replacement
-XX:+EliminateAllocations
```

---

## 2. Lock Optimization via Mark Word (Biased → Thin → Fat Locking)

The **Mark Word** (8 bytes in the object header) is a multipurpose field — its meaning changes based on the object's locking state.

### Lock State Machine

```
Object created
      │
      ▼
[Biased Lock] ──contention──► [Thin Lock] ──heavy contention──► [Fat Lock]
                                                                      │
                                                                 (OS Mutex)
```

### a) Biased Locking (Epoch-based, removed in JDK 15+)

```
Mark Word: [ Thread ID | Epoch | Age | 1 | 01 ]
```

- Assumption: most locks are acquired by the **same thread repeatedly**
- First acquisition: thread ID is "biased" into Mark Word — no CAS needed
- Re-entry: just compare thread ID — single read, no atomic op
- Revocation needed only on actual contention (expensive — requires STW safepoint)

> Removed in JDK 15 (`-XX:-UseBiasedLocking`) — modern apps have too much contention for the revocation overhead to be worth it.

### b) Thin Lock (Lightweight Lock / Stack Lock)

```
Mark Word: [ Pointer to Lock Record on Stack | 00 ]
```

- Used when **one thread at a time** enters the `synchronized` block but contention is possible
- A **Lock Record** is pushed onto the acquiring thread's stack
- The Mark Word is updated via **CAS (Compare-And-Swap)** — no OS involvement
- Re-entrant: counter in Lock Record is incremented
- Fast — entirely in user space, no system call

### c) Fat Lock (Heavyweight / Inflated Lock)

```
Mark Word: [ Pointer to Monitor Object | 10 ]
```

- Used when **multiple threads contend** for the same lock
- JVM inflates to a **Monitor object** (`ObjectMonitor` in HotSpot)
- Contains: owner thread, entry queue, wait set
- Uses OS-level **mutex/futex** — threads genuinely block (context switch)
- `Object.wait()` / `notify()` only work at this level

### Lock Inflation Flow in HotSpot:

```
Thread A holds thin lock
Thread B tries to acquire → CAS fails → spin briefly
        → still fails → inflate to fat lock → block on OS mutex
Thread A releases → sees fat lock → unparks Thread B
```

---

## 3. TLAB (Thread-Local Allocation Buffer) & Overflow

### Why TLAB exists

Without TLAB, every `new` in a multi-threaded app would require a **CAS on the shared Eden pointer** — serializing all allocations across threads.

### How TLAB works

```
Eden Space
┌─────────────────────────────────────────────────┐
│  Thread-1 TLAB  │  Thread-2 TLAB  │  Free Eden  │
│  [   used   |→] │  [   used   |→] │             │
└─────────────────────────────────────────────────┘
```

- Each thread gets a **private chunk** of Eden at JVM startup or on demand
- Allocations within TLAB: just bump the pointer — no synchronization, no CAS
- The pointer `top` advances; allocation fails when `top >= end`

### TLAB Overflow — what happens?

When the current TLAB is full:

```
Object size > (TLAB end - TLAB top)
         │
         ▼
  Is object large? ──Yes──► Allocate directly in Eden (CAS)
         │                   or Old Gen if > -XX:PretenureSizeThreshold
        No
         ▼
  Retire current TLAB (fill remainder with dummy object for GC walkability)
         │
         ▼
  Request new TLAB from Eden
         │
  Is Eden full? ──Yes──► Trigger Minor GC
         │
        No
         ▼
  Allocate in new TLAB
```

### Key TLAB JVM flags:

```bash
-XX:TLABSize=<bytes>          # Initial TLAB size
-XX:+ResizeTLAB               # JVM auto-tunes TLAB size (default: on)
-XX:TLABWasteTargetPercent=1  # Max % of TLAB wasted on overflow before resize
-XX:+PrintTLAB                # Log TLAB activity
```

### TLAB size tuning insight:

- **Too small** → frequent overflows → more CAS on shared Eden → contention
- **Too large** → wasted space per thread, reduces Eden for others
- JVM auto-resizes based on **allocation rate per thread** — usually hands-off

---

## 4. GC Interaction — Object Lifecycle in Generational Heap

### Generational Hypothesis

> Most objects die young. A few live long.

Java's GC is designed around this — short-lived objects never leave Eden.

### Heap Structure (G1GC / Parallel GC)

```
Heap
├── Young Generation
│   ├── Eden Space        ← new objects born here
│   ├── Survivor Space S0 (From)
│   └── Survivor Space S1 (To)
└── Old Generation (Tenured)
    └── Long-lived objects
```

### Object Lifecycle Step-by-Step

#### Step 1: Born in Eden

```java
MyClass obj = new MyClass(); // → Eden via TLAB
```

GC Age = 0

#### Step 2: Minor GC (Young GC) triggered when Eden fills

- **Mark**: Find live objects in Young Gen (GC roots → reachable objects)
- **Copy**: Live objects **copied** to Survivor Space S1 (or S0)
  - Dead objects in Eden → instantly reclaimed (no sweep needed)
  - GC Age incremented by 1 for survivors

```
Before Minor GC:          After Minor GC:
Eden: [A][B][C][D]        Eden: [ empty ]
S0:   [E]                 S0:   [ empty ]
S1:   [ empty ]           S1:   [A][C][E]  (B,D were dead)
                                 age: 1,1,2
```

#### Step 3: Survivor Space bouncing

Objects alternate between S0 and S1 on each Minor GC, age incrementing each time.

```
Minor GC 1: Eden → S1,  S0 → S1  (S0 cleared)
Minor GC 2: Eden → S0,  S1 → S0  (S1 cleared)
```

#### Step 4: Tenuring (Promotion to Old Gen)

When GC Age reaches **tenuring threshold** (default: 15):

```bash
-XX:MaxTenuringThreshold=15
```

Object is **promoted** to Old Generation.

Also promoted early if:
- Survivor space is **too full** (dynamic tenuring threshold auto-adjusts)
- Object is **too large** at creation (`-XX:PretenureSizeThreshold`)

#### Step 5: Major / Full GC

Old Generation fills up → triggers **Major GC** (or Full GC):

- Much more expensive — entire heap or Old Gen is collected
- Causes longer **Stop-The-World** pauses
- G1GC and ZGC are designed to make this concurrent and low-pause

### GC Algorithm Comparison

| GC           | Young | Old               | Pause    | Use Case                      |
|--------------|-------|-------------------|----------|-------------------------------|
| **Serial**   | Copy  | Mark-Compact      | STW      | Single-threaded, small heaps  |
| **Parallel** | Copy  | Mark-Compact      | STW      | Throughput-focused            |
| **G1GC**     | Copy  | Concurrent Mixed  | Short STW| Balanced, default JDK 9+      |
| **ZGC**      | Copy  | Concurrent        | Sub-ms   | Low latency, large heaps      |
| **Shenandoah**| Copy | Concurrent        | Sub-ms   | Low latency                   |

---

## Complete Flow — Interview-Level Summary

```
new MyClass()
      │
      ├─► JIT checks: does object escape?
      │       No  → Stack allocate / Scalar replace (no heap, no GC)
      │       Yes → Continue to heap
      │
      ├─► TLAB has space?
      │       Yes → Bump pointer (nanoseconds, no sync)
      │       No  → Retire TLAB, get new one / CAS on Eden / trigger Minor GC
      │
      ├─► Object Header written:
      │       Mark Word → biased lock or unlocked state
      │       Klass Pointer → Metaspace class metadata
      │
      ├─► Fields zero-initialized → Constructor executes
      │
      └─► Object lives in Eden, Age=0
              │
          Minor GC survival → Survivor spaces, Age++
              │
          Age == 15 → Promoted to Old Gen
              │
          Old Gen fills → Major GC
```

### Key Insight

> **`new` is designed to be cheap** — TLAB makes it near-free in the common case,
> Escape Analysis can eliminate it entirely, and the generational GC ensures
> short-lived objects are collected at minimal cost.
