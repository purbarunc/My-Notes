# Interview Questions - Level 1 (Basics) - Java

---

## #1. What is Java?

### What is the interviewer looking for?
- Familiarity with the language, language constructs, its syntax, and some of the feature benefits.
- Compare it with other previous programming languages of that time.

### Answer

**Key features of Java:**

- **Object Oriented Programming Language**
- **Portable** - One of the biggest promises of Java was the write-once-and-run-anywhere concept. We basically can write our code, compile it once and run it on various different machines. This was not possible in the programming world at the time Java was introduced with C/C++, where we had to compile with different compilers and write different code to make our code work with different processor architectures. Java has none of these problems.
- **Server Side applications** - It is one of the most popular languages for building server-side applications.

---

## #2. What are some of the features of Java?

### What is the interviewer looking for?
- Awareness of what the language has to offer.
- Comparison with other languages.
- Important language characteristics.

### Answer

There are 5 key characteristics or goals that the creators of Java set to achieve:

1. **Simple, Object Oriented, and familiar** - It was created at the time when C and C++ were ruling the programming landscape. Hence Java has object-oriented syntax features which were similar to that of C/C++.

2. **Robust and Secure** - It is because of the Java Virtual Machine and the associated concepts. Java does not run on the native processor. It runs on a virtual machine called the JVM and because of this, there is a lot of robustness and security built around Java. Our code does not have direct access to machine-level code, since there are many checks and bounds of JVM. Hence it is more secure than similar C/C++ applications.

3. **Architecture-neutral and portable** - This is the key to the write-once-and-run-anywhere characteristic of Java. Since the code runs in a VM, the VM forms a layer on top of the actual processor which is running the code. A compiled byte code can run on any machine as long as there is a JVM set up in that machine. Hence Java is architecture-neutral and portable.

4. **High performance** - It was designed to be high-performance. Initially, it was not quite meeting this promise due to the extra layer which handles portability. But these days the JVM has improved so much that Java runs just as fast as C/C++ programs. There have been instances where it actually runs faster than native code generation.

5. **Interpreted, threaded, and dynamic** - Byte code is interpreted, hence it's an interpreted runtime.

---

## #3. What is a JVM?

### What is the interviewer looking for?
- Difference between JVM and JRE.
- Characteristics of JVM.
- The role it plays in the execution.

### Answer

- **JVM** stands for **Java Virtual Machine**.
- It's a runtime VM in which a Java program is run. When we run a Java program it does not run on our bare-bone machine directly; instead, it creates a virtual machine which is a kind of abstraction that the runtime creates.
- Takes the compiled byte code and executes it — hence the input to the JVM is the byte code.
- JVM has a specification that outlines how it should work. There are multiple JVM implementations available. We can run Java byte code on any of them and it will work exactly the same way because all those JVMs follow a standard specification.
- The JVM concept is essential in making Java platform agnostic (write-once-and-run-anywhere). Since Java is not directly running on the hardware but on this abstraction layer, as long as someone can build a JVM according to the specification, it doesn't matter what the underlying architecture is — our Java application will work seamlessly.

---

## #4. What is a JRE?

### What is the interviewer looking for?
- Difference between JVM and JRE.
- Characteristics of JRE.
- The role it plays in the execution.

### Answer

- **JRE** stands for **Java Runtime Environment**.
- It is a set of software that we install in our machine which together works to run a Java application.
- **Consists of:**
  - Class loader
  - JVM
  - Libraries and utilities which assist the code to execute
- JRE orchestrates activities between these software elements.

---

## #5. What is a JDK?

### What is the interviewer looking for?
- What is the difference between JRE and JDK.
- Characteristics of JDK.
- The role it plays in the development workflow.

### Answer

- **JDK** stands for **Java Development Kit**.
- It is a set of tools to help developers write Java programs. If we just want to execute Java code then we need a JRE, but if we want to write and publish Java applications then we need the JDK.
- JDK comes bundled with the JRE (since we need to run what we develop).
- It is also based on Java language specifications. There are multiple JDK implementations, but as long as it's a certified JDK, it ensures the code written in Java will result in byte code that implements the same functionality as intended.
- **Includes:**
  - **Compiler** - converts Java code to byte code
  - **Class Libraries** - forms the core native API for Java
  - **Utilities** - Loading, Linking, or packaging into a jar

---

## #6. What is Java Bytecode?

### What is the interviewer looking for?
- Understanding how it is created / the compilation process.
- How it supports portability.
- Relation with JVM.

### Answer

- **Instruction set for JVM** - Bytecode is the instruction set that the virtual machine executes. We compile code to bytecode; the bytecode forms the instruction set for the JVM layer.
- It is **generated by the Java compiler** as part of the compilation process — input is the Java source code, output is the Java bytecode.
- JRE gives this to JVM. JVM then takes it and executes it.
- Java Bytecode **cannot be run natively** on a machine. We need the JRE in order to run it.
- **Bytecode is consistent across machines**, but JVM implementations may vary. This is how Java is portable — we can take the bytecode and take it to a completely different machine and have that JVM execute it. It will run exactly the same way as long as the JVM meets the specifications. Hence, it is platform agnostic.
- This enables the **"write-once-run-anywhere"** feature of Java.

**Summary:**
| Component | Purpose |
|-----------|---------|
| JDK | Software used for building Java Applications |
| JRE | Software used for running Java Applications |
| JVM | Abstract virtual machine that the JRE spins up every time it needs to run a Java Application |

---

## #7. What is the difference between Path and Classpath?

### What is the interviewer looking for?
- Understanding of system variables.
- Understanding of the classpath concept.
- Why do we need to set these variables.

### Answer

Both are **environment/system variables** that we set in our operating system.

- **Path** - An operating system-specific variable that influences what binaries and executables are available for runtime when you are running on a command prompt or a terminal. When we open the terminal or command prompt, we can access executables from all directories listed in the `PATH` variable. This has nothing to do with Java specifically.

- **Classpath** - Not operating system specific. This is actually a **Java construct**. Classpath is used to indicate where our compiled classes and jars are placed. It's a path to the classes for the execution of a particular application. It can include multiple locations. Classpath is set for execution or for compilation — a compiler needs to know where your jars and libraries are so that compilation can happen. It can change from compilation to compilation and from execution to execution, like a command line argument.

---

## #9. What are the different memory areas allocated by the JVM?

### What is the interviewer looking for?
- The distinction between the important ones — stack and heap.
- Class area and native area are a bonus!

### Answer

### Heap
- **Space for objects in memory** - The heap is where JVM allocates memory for objects. Any time we create a new instance of an object by calling a constructor, the space is allocated in the heap.
- **"Global"** - Object memory can be thought of as a global space. Different methods and different threads might all reference the same shared space in the heap. Hence it is like a shared/global space.
- It is the **largest of all the memory spaces**.

### Stack
- **Holds thread-level data** - A stack holds thread-level data. When a method is being executed, there is a thread running where that method is executed, along with local variables specific to that thread.
- **Local variables and object references** - Object references also go into the stack, but they point to objects in the heap.
- **Call frames for each method execution** - For each method execution, call frames or execution frames get created. When a method returns, that frame gets taken out of the stack. Stacks are short-lived.

### Code Area (Metaspace)
- **Stores bytecode and JIT info** - This is where the JVM allocates space for the bytecode or the just-in-time compilation information. It also stores any information required for just-in-time compilation.
- This is internal to the implementation of the JVM. It's related to the execution of the code, not the code itself — hence called the **metaspace**.

### Implementation / Native Area
- **Registers and C implementation stacks** - Native area refers to the low-level C-level code of the JVM implementation. A lot of JVM implementations are written in C and C++. These are JVM-level constructs and may vary between different JVM implementations.

---

## #10. What's the difference between PermGen and Metaspace?

### What is the interviewer looking for?
- Understanding of the Java 8 changes in memory management.
- Awareness of Metaspace.

### Answer

Both **PermGen** and **Metaspace** are memory areas in the JVM where it loads the bytecode.

- **Before Java 8:** There was a memory area called **Permanent Generation (PermGen)**. It had a **fixed max size** (configurable). When something was loaded into PermGen, it remained there as long as the program was executed — hence the name "permanent generation."

- **Java 8 onwards:** PermGen was replaced by **Metaspace**. Metaspace **expands as needed by default** until it runs out of system memory.

- We can assign a max space to Metaspace using the `maxMetaSpaceSize` configuration — making it behave like PermGen. When the max space is approached, the JVM starts cleanup: it garbage-collects dead classes and bytecode loaded in Metaspace. So technically, by using this configuration, we can make Metaspace behave like PermGen.

- The key difference: with PermGen, once something was loaded it was permanent. With Metaspace, class loading and unloading is **dynamic**, and the JVM manages it seamlessly without a hard limit by default.

---

## #11. What is Garbage Collection?

### What is the interviewer looking for?
- Awareness of the process.
- Some details of how it works.
- Benefits and drawbacks.

### Answer

- **Removing unused and orphaned objects from the heap** - It is the automated process of removing unused and orphaned objects from the heap. An object with no references pointing to it is an orphaned object that claims space but is no longer needed. Garbage collection reclaims this memory.
- **Automated process** - Does not need a programmer's intervention.
- **Can be triggered programmatically** - In rare cases, we can trigger garbage collection using `System.gc()` or `Runtime.getRuntime().gc()`. This is not recommended unless we are very sure about it.
- **Generation-based approach** - Uses a generation-based strategy to manage memory.

**Benefits:**
- No manual memory management required.

**Drawbacks:**
- **Performance intrusion** - When Garbage Collection runs, there is a small impact on execution. System resources are split between business logic and garbage collection logic. Some strategies halt the execution of business logic until cleanup is complete, though typically for only a fraction of a second.

---

## #12. What is JIT Compilation?

### What is the interviewer looking for?
- Familiarity with how JVM works — understanding of the concept of interpretation rather than direct execution.
- Byte code vs Native code.

### Answer

- **JIT** stands for **Just-In-Time** compilation.
- The JVM **selectively converts certain bytecode instructions into native code** depending on usage and how the JVM perceives it will improve overall performance.
- The JVM doesn't automatically transform all the bytecode into native code, because that would result in a huge performance penalty. Instead, it selectively decides — for example, if a loop needs to run 50,000 times, the JVM may compile those instructions down to native processor-level code.
- This native code depends on the processor and hardware architecture it's running on, so it almost feels like that particular portion is being compiled for the specific processor architecture.
- The JVM selectively does this depending on usage, with the intent to optimize and ensure that code runs fast.

---

## #42. What is Abstraction?

### What is the interviewer looking for?
- Concept understanding.
- How it affects the class design.
- Interface vs implementation.

### Answer

- **Abstraction** is the object-oriented design principle of **separating the interface from the implementation** so that the consumer/client is only concerned with the interface. The external party interacts only with the interface, not with the implementation.

- **Real World Example:** Buttons on electric devices vs internal circuit board — A coffee maker has several buttons for many functionalities. These buttons are like interfaces. A user will never interact with the internal circuit board (the implementation) to make coffee. The implementation is abstracted away.

- Manifests in class design using **interfaces**. The concept of interfaces is precisely for building abstraction.
- Abstraction can be enforced using **encapsulation**.

> **Note:** Abstract classes are not meant for abstraction — don't be misled by the name. Abstract classes are for consolidating common things that need to exist in multiple classes. They are a convenience feature for authors of a class hierarchy.

---

## #43. What is Encapsulation?

### What is the interviewer looking for?
- Concept understanding.
- How it affects the class design.

### Answer

- **Encapsulation** is the process of **restricting access to the inner implementation details of a class**. It enforces abstraction concepts by not just hiding but by guaranteeing the internals aren't exposed.

- **Real-World Example:** Sealed devices prevent people from messing with the internals.

- Manifests in class design using **access modifiers** — like `private`.
