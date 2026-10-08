# Why Service and Repository Classes are Singletons in Spring Boot

In Spring Boot, beans managed by the Spring IoC (Inversion of Control) container are **singleton by default** — meaning one instance is created per application context. Here's why this design makes sense for Services and Repositories:

---

## 1. Statelessness

Service and Repository classes are designed to be **stateless** — they don't store user-specific or request-specific data as instance fields. They only contain:
- Business logic (Services)
- Database interaction logic (Repositories)

Since they hold no mutable state, sharing a single instance across all callers is safe and correct.

---

## 2. Performance & Memory Efficiency

Creating a new object for every request is expensive:
- Object instantiation takes CPU cycles
- More objects mean more GC (Garbage Collection) pressure
- Singletons avoid this overhead entirely — one instance handles all requests

---

## 3. Thread Safety (by design)

Because these classes are stateless, they are inherently thread-safe. Multiple threads can invoke methods on the same instance simultaneously without risk of data corruption.

> **Important:** If you accidentally store state in a `@Service` field (e.g., `private User currentUser`), you break this assumption and introduce race conditions.

---

## 4. Dependency Injection Works Best with Singletons

Spring's DI container wires dependencies at startup. If every bean were a new instance per request, Spring would have to re-resolve and re-inject the entire dependency graph on every call — which is wasteful and complex.

Singletons allow Spring to wire everything **once** at startup and reuse the graph.

---

## 5. Resource Management

Repositories often hold references to shared resources like:
- Database connection pools (`DataSource`)
- `EntityManager` factories

These are expensive to create. A singleton Repository holds one reference to the pool and reuses it across all calls.

---

## Scope Summary

| Scope | Instances | Use Case |
|---|---|---|
| `singleton` (default) | 1 per context | `@Service`, `@Repository`, `@Component` |
| `prototype` | New per injection | Stateful helper objects |
| `request` | New per HTTP request | Web-layer, request-scoped beans |
| `session` | New per HTTP session | User session data |

---

## Key Takeaway

> Singletons work for Services and Repositories **because these classes are stateless**. The singleton scope is not an arbitrary choice — it's the correct match for stateless, shared logic. If a bean needs state, you'd use `@Scope("prototype")` or `@Scope("request")` instead.

---

## What Happens When You Inject a Prototype Bean into a Singleton?

This is a classic Spring "gotcha."

### The Problem

```java
@Component
@Scope("prototype")
public class PrototypeBean {
    // supposed to be a new instance every time
}

@Service  // singleton by default
public class SingletonService {

    @Autowired
    private PrototypeBean prototypeBean; // INJECTED ONCE AT STARTUP

    public void doWork() {
        prototypeBean.someMethod(); // always the SAME instance!
    }
}
```

**What you expect:** A new `PrototypeBean` instance every time `doWork()` is called.

**What actually happens:** Spring injects `PrototypeBean` **once** when `SingletonService` is created at startup. That single instance is stored in the singleton's field and **never changes** — the prototype effectively behaves like a singleton.

### Why This Happens

The singleton bean is created **once**. At that moment, Spring resolves all its `@Autowired` dependencies and stores references to them. The singleton has no mechanism to ask Spring for a new bean later — it just holds whatever was injected at creation time.

---

### How to Fix It

#### Option 1: `ApplicationContext.getBean()` (manual lookup)

```java
@Service
public class SingletonService implements ApplicationContextAware {

    private ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;
    }

    public void doWork() {
        PrototypeBean fresh = context.getBean(PrototypeBean.class); // new instance every time
        fresh.someMethod();
    }
}
```

#### Option 2: `@Lookup` method injection (cleaner)

```java
@Service
public abstract class SingletonService {

    @Lookup
    public abstract PrototypeBean getPrototypeBean(); // Spring overrides this to return a new instance

    public void doWork() {
        getPrototypeBean().someMethod(); // new instance every call
    }
}
```

Spring subclasses your class at runtime and overrides the `@Lookup` method to call `getBean()` internally.

#### Option 3: `ObjectProvider` (most modern / recommended)

```java
@Service
public class SingletonService {

    @Autowired
    private ObjectProvider<PrototypeBean> prototypeBeanProvider;

    public void doWork() {
        PrototypeBean fresh = prototypeBeanProvider.getObject(); // new instance every time
        fresh.someMethod();
    }
}
```

---

### Fix Summary

| Approach | Problem | Fix |
|---|---|---|
| `@Autowired` prototype into singleton | Gets injected **once**, acts like singleton | Don't do this |
| `ApplicationContext.getBean()` | Works, but couples to Spring API | Acceptable |
| `@Lookup` | Clean, Spring handles it | Recommended |
| `ObjectProvider<T>` | Lazy, clean, testable | Most recommended (Spring 4.3+) |

---

### Key Rule

> If a **shorter-lived** bean (prototype, request) is needed inside a **longer-lived** bean (singleton), you cannot use plain `@Autowired`. You must use a mechanism that **defers the lookup** until the method is actually called.
