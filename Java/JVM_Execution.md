# How the JVM Executes a Java Program Internally

---

## 1. Compilation (Before JVM)

```
Source Code (.java) → javac compiler → Bytecode (.class)
```

The Java compiler (`javac`) translates your `.java` source into **platform-neutral bytecode** stored in `.class` files. This is NOT machine code — it's an intermediate representation.

---

## 2. Class Loading

When you run `java MyProgram`, the JVM starts the **Class Loading** phase via the **ClassLoader subsystem**:

```
Bootstrap ClassLoader   →  loads core Java classes (java.lang.*, etc.)
Extension ClassLoader   →  loads JDK extension classes
Application ClassLoader →  loads your application's classes
```

Each ClassLoader follows the **delegation model** — it first asks its parent before loading a class itself (prevents tampering with core classes).

**Three steps per class:**
- **Loading** — reads the `.class` file bytes into memory
- **Linking** — verifies bytecode, allocates memory for static fields (set to defaults), resolves symbolic references
- **Initialization** — runs static initializers and sets static field values

---

## 3. JVM Runtime Data Areas (Memory Layout)

Once loaded, the JVM manages several memory regions:

| Area | Scope | Contents |
|---|---|---|
| **Method Area** (Metaspace in Java 8+) | Shared | Class metadata, static variables, constant pool |
| **Heap** | Shared | All object instances and arrays |
| **JVM Stack** | Per-thread | Stack frames (local vars, operand stack, return address) |
| **PC Register** | Per-thread | Address of current executing instruction |
| **Native Method Stack** | Per-thread | For native (C/C++) method calls |

---

## 4. Execution Engine

This is where bytecode actually runs:

```
Bytecode
   |
   ├──► Interpreter  (executes instruction-by-instruction, slow)
   |
   └──► JIT Compiler (Just-In-Time)
           |
           ├── Detects "hot" code (frequently executed methods/loops)
           ├── Compiles bytecode → native machine code at runtime
           └── Caches native code for subsequent calls (fast)
```

The JIT uses **profiling counters** — methods called above a threshold get compiled. HotSpot JVM uses two tiers:
- **C1 (Client Compiler)** — quick compile, light optimization
- **C2 (Server Compiler)** — aggressive optimization (inlining, escape analysis, loop unrolling)

Modern JVMs use **Tiered Compilation** (C1 → profile → C2).

---

## 5. The Execution Loop (Bytecode Interpretation)

Each method call creates a **stack frame** containing:
- **Local variable array** — method arguments + local vars
- **Operand stack** — working area (like CPU registers, but stack-based)
- **Reference to constant pool** — symbolic references for the class

Example for `int result = a + b;`:
```
iload_1      // push 'a' onto operand stack
iload_2      // push 'b' onto operand stack
iadd         // pop both, add, push result
istore_3     // pop result into local variable 'result'
```

---

## 6. Garbage Collection

The JVM automatically manages heap memory:

```
Heap Layout (Generational GC):

Young Generation          Old Generation    Metaspace
┌──────────────────┐     ┌──────────────┐  ┌──────────┐
│ Eden │ S0 │  S1  │ →   │   Tenured    │  │  Class   │
│      │    │      │     │   Objects    │  │ Metadata │
└──────────────────┘     └──────────────┘  └──────────┘
```

- **Minor GC** — collects Eden + Survivor spaces (frequent, fast)
- **Major/Full GC** — collects Old Generation (infrequent, slower)
- Objects that survive enough Minor GCs get **promoted** to Old Generation

---

## 7. Entry Point

The JVM looks for:
```java
public static void main(String[] args)
```
This must be `public` (accessible), `static` (no instance needed), `void` (returns nothing). The JVM calls it directly without creating an object.

---

## Full Picture

```
.java → [javac] → .class (bytecode)
                       |
                  [ClassLoader]
                       |
              [JVM Memory Areas]
               Heap / Stack / etc.
                       |
           [Execution Engine]
           Interpreter + JIT Compiler
                       |
              Native Machine Code
                       |
                   CPU executes
                       |
              [GC] manages heap
```

The key insight: Java's "write once, run anywhere" works because bytecode is **hardware-agnostic**, and the JVM on each platform translates it to native instructions at runtime via the JIT.
