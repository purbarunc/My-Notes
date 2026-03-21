# JVM Architecture

## Virtual Machine

It is a software simulation of a machine that can perform operations like a physical machine.

There are 2 types of virtual machines:

1. Hardware/System based
2. Application/Process-based

### Hardware/System based Virtual Machine

- It provides several logical systems on the same computer with strong isolation from each other i.e on one physical machine we are defining multiple logical machines.
- The main advantage of hardware-based virtual machines is hardware resource sharing and improved utilization of hardware resources. eg. KVM (Kernel-based VM for Linux System), VMware, etc.

### Application/Process-based Virtual Machine

These VMs act as runtime engines to run particular programming language applications.

Examples:
- a. JVM acts as a runtime engine to run Java-based applications
- b. PVM (Parrot Virtual Machine) acts as a runtime engine to run Perl based applications
- c. CLR (Common Language Runtime) acts as a runtime engine to run .NET-based applications

---

## JVM

JVM is a part of JRE and it is responsible for loading and running java class files.

### Basic Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        JVM Architecture                         │
│                                                                 │
│  .class file ──────────────► Class Loader System               │
│                                      │                         │
│              ◄───────────────────────┘                         │
│              ▼                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Various Memory Areas of JVM                │   │
│  │                                                         │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │   │
│  │  │  Method  │ │   Heap   │ │  Stack   │ │    PC    │  │   │
│  │  │   Area   │ │   Area   │ │   Area   │ │ Register │  │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │   │
│  │                                   ┌────────────────┐   │   │
│  │                                   │ Native Method  │   │   │
│  │                                   │     Stack      │   │   │
│  │                                   └────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│              │                                                  │
│              ▼                                                  │
│  ┌───────────────────┐    ┌───────────────┐   ┌─────────────┐  │
│  │  Execution Engine │◄──►│  Java Native  │◄──│   Native    │  │
│  │                   │    │Interface (JNI)│   │Method Libs  │  │
│  └───────────────────┘    └───────────────┘   └─────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ClassLoader Sub System

ClassLoader Sub System is responsible for the following 3 activities:

1. Loading
2. Linking
3. Initialization

### ClassLoader Flow Diagram

```
                    ┌─────────┐
                    │  Verify │
                    └────┬────┘
                         │
┌─────────┐              ▼          ┌────────────────┐
│ Loading │──────►  ┌─────────┐ ───►│ Initialization │
└─────────┘         │ Prepare │     └────────────────┘
                    └────┬────┘
                         │
                    ┌────▼────┐
                    │ Resolve │
                    └─────────┘
                    └──── Linking ────┘
```

### Loading

Loading means reading class files and storing corresponding binary data in the *method area*.

For each class file JVM will store corresponding information in the method area:

1. Fully Qualified name of the class
2. Fully Qualified Name of the immediate parent class
3. Methods information
4. Variables information
5. Constructor information
6. Modifiers information
7. Constant pool information, etc.

After loading the `.class` file, JVM immediately creates an object for that loaded class in heap memory of type `java.lang.class`. The class `Class` object can be used by programmers to get class-level information like methods information, variable information, constructors information, etc. using the reflection framework.

**Example:**

```java
import java.lang.reflect.*;

class Student {
    public String getName() {
        return null;
    }
    public int getMarks() {
        return 10;
    }
}

class Test {
    public static void main(String args[]) {
        int count = 0;
        Class c = Class.forName("Student"); // Load the class
        Method[] m = c.getDeclaredMethods();
        System.out.println("Number of methods:" + c.length()); // Prints total no. of methods
        for (Method m1 : m) {
            System.out.println(m1.getName()); // Prints all method names of Student class
        }
    }
}
```

For every loaded type, only one class object will be created even though we are using the class multiple times in our program.

**Example:**

```java
class Student {
    public String getName() {
        return null;
    }
    public int getMarks() {
        return 10;
    }
}

class Test {
    public static void main(String args[]) {
        Student s1 = new Student();
        Class c1 = s1.getClass();
        Student s2 = new Student();
        Class c2 = s2.getClass();
        System.out.println(c1.hashcode() == c2.hashcode()); // true
        System.out.println(c1 == c2); // true
    }
}
```

---

### Linking

Linking consists of 3 activities:

1. Verification *(or, Verify)*
2. Preparation *(or, Prepare)*
3. Resolution *(or, Resolve)*

#### Verification

- It is the process of ensuring that the binary representation of a class is structurally correct or not i.e JVM will check whether `.class` is generated by a valid compiler or not and properly formatted or not.
- Internally ByteCode verifier is responsible for this activity. ByteCode verifier is the part of Classloader sub-system.
- If verification fails then we will get a runtime exception saying `java.lang.VerifyError`

#### Preparation

In this phase, JVM will allocate memory for class-level static variables and assign default values.

> **Note:** In the Initialization phase, original values will be assigned to the static variables.

#### Resolution

It is the process of replacing symbolic names in our program with original memory references from the method area.

**Example:**

```java
class Test {
    public static void main(String args[]) {
        String s = new String("Test");
        Student s1 = new Student();
    }
}
```

For the above class, class loader loads `Test.class`, `String.class`, `Student.class` and `Object.class`. The names of these classes are stored in a constant pool of Test class. In the resolution phase, these names are replaced with original memory level references from the method area.

---

### Initialization

In this phase, all static variables are assigned with original values and static blocks will be executed from parent to child and from top to bottom.

While loading, linking, and initialization if any error occurs then we will get a runtime exception saying `java.lang.LinkageError`

---

## ClassLoaders

ClassLoaders subsystem contains the following 3 types of class loaders:

1. Bootstrap/Primordial class loader
2. Extension class loader
3. Application/System class loader

### Bootstrap Class Loader

- The bootstrap class loader is responsible to load core java API classes i.e. the classes present in `rt.jar`
- The bootstrap class loader is responsible to load classes from the bootstrap classpath which is `JDK\JRE\lib`
- The bootstrap class loader is by default available with every JVM and it is implemented in native languages C/C++ and not implemented in Java

### Extension Class Loader

- Extension class loader is the child class of bootstrap class loader
- The extension class loader is responsible to load classes from the extension classpath which is `JDK\JRE\lib\ext`
- The extension class loader is implemented in Java and the corresponding `.class` file is `sun.misc.Launcher$ExtClassLoader.class`

### Application Class Loader

- Application class loader is the child class of extension class loader
- This class loader is responsible to load classes from application class path
- It internally uses environment variable class path
- The application class loader is implemented in Java and the corresponding `.class` file is `sun.misc.Launcher$AppClassLoader.class`
