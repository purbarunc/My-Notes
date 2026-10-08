# Underlying Data Structures Behind Java Collections

A rundown of the underlying data structures for the main Java collections (`java.util`), plus the concurrent ones in `java.util.concurrent`.

## List

| Class | Underlying structure |
|---|---|
| `ArrayList` | Resizable array (`Object[]`), amortized O(1) append, O(n) insert/remove in middle |
| `LinkedList` | Doubly-linked list of `Node` objects, O(1) insert/remove at ends, O(n) random access |
| `Vector` | Resizable array (like `ArrayList`) but all methods are `synchronized` |
| `Stack` | Extends `Vector` — resizable array, synchronized (legacy, prefer `Deque`) |
| `CopyOnWriteArrayList` | Immutable array snapshot; every mutation copies the entire array, reads are lock-free |

## Set

Sets are almost always backed by a `Map` (elements are keys, values are a dummy `PRESENT` object):

| Class | Underlying structure |
|---|---|
| `HashSet` | `HashMap` (hash table) |
| `LinkedHashSet` | `LinkedHashMap` (hash table + doubly-linked list for insertion order) |
| `TreeSet` | `TreeMap` (red-black tree), O(log n) operations, sorted |
| `EnumSet` | Bit vector — a single `long` (or `long[]` for >64 enum values) |
| `CopyOnWriteArraySet` | `CopyOnWriteArrayList` |

## Map

| Class | Underlying structure |
|---|---|
| `HashMap` | Hash table: array of buckets, each bucket is a linked list that **treeifies into a red-black tree** when ≥8 entries (`TREEIFY_THRESHOLD`) and untreeifies below 6 |
| `LinkedHashMap` | `HashMap` + doubly-linked list through entries, maintains insertion (or access) order — basis for LRU caches |
| `TreeMap` | Red-black tree (self-balancing BST), sorted by key |
| `Hashtable` | Hash table with every method synchronized (legacy) |
| `IdentityHashMap` | Hash table using `System.identityHashCode` and `==` comparison, linear probing |
| `WeakHashMap` | Hash table with `WeakReference` keys, entries auto-removed after GC |
| `EnumMap` | Dense array indexed by enum ordinal — very compact and fast |
| `Properties` | Extends `Hashtable` — synchronized hash table |

## Queue / Deque

| Class | Underlying structure |
|---|---|
| `ArrayDeque` | Resizable **circular array** — faster than `Stack`/`LinkedList` for stack/queue use |
| `PriorityQueue` | Binary **min-heap** in an array (parent at `(i-1)/2`, children at `2i+1`, `2i+2`) |
| `LinkedList` | Doubly-linked list (also implements `Deque`) |
| `PriorityBlockingQueue` | Binary heap (unbounded, blocking) |

## Concurrent Collections (`java.util.concurrent`)

| Class | Underlying structure |
|---|---|
| `ConcurrentHashMap` | Hash table like `HashMap` (buckets → list → red-black tree), but with **CAS + per-bin `synchronized`** locking; no segments since Java 8 (pre-8 it used segmented locking) |
| `ConcurrentLinkedQueue` | Non-blocking linked queue using CAS (Michael-Scott algorithm) |
| `ConcurrentLinkedDeque` | Non-blocking doubly-linked deque using CAS |
| `ConcurrentSkipListMap` / `ConcurrentSkipListSet` | **Skip list** — lock-free sorted structure, O(log n), alternative to `TreeMap` |
| `LinkedBlockingQueue` / `LinkedBlockingDeque` | Linked nodes with separate take/put locks ("two-lock queue"), optionally bounded |
| `ArrayBlockingQueue` | Bounded **circular array** with a single lock + two conditions |
| `DelayQueue` | `PriorityQueue` ordered by delay expiry, blocking |
| `SynchronousQueue` | No storage — hand-off structure (dual stack/queue of waiting threads) |

## Quick Mental Model

- **Array-backed** (cache-friendly, fast index access): `ArrayList`, `Vector`, `ArrayDeque`, `EnumMap`, `ArrayBlockingQueue`
- **Linked-node-backed** (fast head/tail mutation): `LinkedList`, `LinkedBlockingQueue`
- **Hash table**: `HashMap`/`HashSet` family — buckets that become red-black trees under collision pressure
- **Balanced tree** (sorted, O(log n)): `TreeMap`/`TreeSet` — red-black tree
- **Skip list** (concurrent + sorted): `ConcurrentSkipList*`
- **Heap**: `PriorityQueue`, `PriorityBlockingQueue`, `DelayQueue`
- **Bit vector**: `EnumSet`
- **Copy-on-write**: `CopyOnWriteArrayList`/`CopyOnWriteArraySet`

The non-obvious points worth remembering: `HashSet` is just a `HashMap` wrapper, `HashMap` buckets treeify into red-black trees (Java 8+), and `ConcurrentSkipListMap` is the sorted concurrent map — there's no concurrent red-black tree in the JDK.
