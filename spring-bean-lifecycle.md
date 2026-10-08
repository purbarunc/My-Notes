# Spring Bean Lifecycle

The Spring Bean Lifecycle describes the series of steps a bean goes through from its instantiation to its destruction within the Spring IoC container.

---

## Overview of the Lifecycle

```
Container Starts
      |
      v
1. Bean Definition Loading
      |
      v
2. Bean Instantiation
      |
      v
3. Dependency Injection (Populate Properties)
      |
      v
4. Aware Interfaces (BeanNameAware, BeanFactoryAware, etc.)
      |
      v
5. BeanPostProcessor - postProcessBeforeInitialization()
      |
      v
6. InitializingBean.afterPropertiesSet() / @PostConstruct / init-method
      |
      v
7. BeanPostProcessor - postProcessAfterInitialization()
      |
      v
8. Bean is Ready for Use
      |
      v
9. DisposableBean.destroy() / @PreDestroy / destroy-method
      |
      v
Container Shuts Down
```

---

## Phase-by-Phase Breakdown

### Phase 1: Bean Definition Loading

Before any bean is created, Spring reads all configuration sources:

- `@Configuration` classes
- XML config files
- `@ComponentScan` paths
- `@Bean` method definitions

These are stored as `BeanDefinition` objects in the `BeanDefinitionRegistry`.

---

### Phase 2: Bean Instantiation

Spring instantiates the bean using one of:

- **Constructor injection** (preferred)
- **No-arg constructor** + setter injection
- **Factory method** (static or instance)

```java
@Component
public class MyService {
    public MyService() {
        System.out.println("1. Bean instantiated via constructor");
    }
}
```

> By default, beans are **Singleton** — instantiated once per container. `Prototype` beans are instantiated each time they are requested.

---

### Phase 3: Dependency Injection (Property Population)

After construction, Spring injects dependencies via:

- `@Autowired` fields
- `@Autowired` setter methods
- Constructor parameters (already done at instantiation if constructor injection)

```java
@Component
public class MyService {
    @Autowired
    private MyRepository repo; // injected after construction

    @Autowired
    public void setRepo(MyRepository repo) {
        this.repo = repo; // or via setter
    }
}
```

---

### Phase 4: Aware Interfaces

Spring calls Aware callbacks to give the bean access to container infrastructure:

| Interface | Method | Provides |
|---|---|---|
| `BeanNameAware` | `setBeanName(String)` | The bean's name in the container |
| `BeanFactoryAware` | `setBeanFactory(BeanFactory)` | The owning `BeanFactory` |
| `ApplicationContextAware` | `setApplicationContext(ApplicationContext)` | The full `ApplicationContext` |
| `EnvironmentAware` | `setEnvironment(Environment)` | The `Environment` object |
| `ResourceLoaderAware` | `setResourceLoader(ResourceLoader)` | For loading classpath resources |

```java
@Component
public class MyService implements BeanNameAware, ApplicationContextAware {

    @Override
    public void setBeanName(String name) {
        System.out.println("Bean name: " + name);
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("ApplicationContext injected");
    }
}
```

> Use Aware interfaces sparingly — they couple your bean to the Spring API.

---

### Phase 5: BeanPostProcessor — `postProcessBeforeInitialization()`

`BeanPostProcessor` is a powerful extension point. Spring calls this **before** the bean's own init method runs.

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("Before init: " + beanName);
        return bean; // return the (possibly wrapped) bean
    }
}
```

This is how Spring implements:
- `@Autowired` processing (`AutowiredAnnotationBeanPostProcessor`)
- `@PostConstruct` / `@PreDestroy` processing (`CommonAnnotationBeanPostProcessor`)
- AOP proxy creation (`AnnotationAwareAspectJAutoProxyCreator`)

---

### Phase 6: Initialization Callbacks

Three mechanisms, called **in this order**:

#### 6a. `@PostConstruct` (JSR-250 — preferred)
```java
@PostConstruct
public void init() {
    System.out.println("@PostConstruct called");
}
```

#### 6b. `InitializingBean.afterPropertiesSet()`
```java
@Component
public class MyService implements InitializingBean {

    @Override
    public void afterPropertiesSet() {
        System.out.println("afterPropertiesSet called");
    }
}
```

#### 6c. `init-method` (XML or `@Bean`)
```java
@Bean(initMethod = "customInit")
public MyService myService() {
    return new MyService();
}

// in MyService:
public void customInit() {
    System.out.println("Custom init-method called");
}
```

> If all three are defined, execution order is: `@PostConstruct` → `afterPropertiesSet()` → `init-method`

---

### Phase 7: BeanPostProcessor — `postProcessAfterInitialization()`

Called **after** all initialization callbacks. This is where Spring typically **wraps beans in AOP proxies**.

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    System.out.println("After init: " + beanName);
    return bean; // may return a proxy instead of the original bean
}
```

> The object returned here is what is stored in the context and injected elsewhere — that's why AOP works transparently.

---

### Phase 8: Bean Ready for Use

The bean is now fully initialized and stored in the singleton cache. It is injected wherever referenced and handles application requests.

---

### Phase 9: Destruction Callbacks (on Container Shutdown)

When `context.close()` or `ctx.registerShutdownHook()` is triggered:

#### 9a. `@PreDestroy` (preferred)
```java
@PreDestroy
public void cleanup() {
    System.out.println("@PreDestroy called");
}
```

#### 9b. `DisposableBean.destroy()`
```java
@Component
public class MyService implements DisposableBean {

    @Override
    public void destroy() {
        System.out.println("DisposableBean.destroy() called");
    }
}
```

#### 9c. `destroy-method` (XML or `@Bean`)
```java
@Bean(destroyMethod = "customDestroy")
public MyService myService() { ... }
```

> Order: `@PreDestroy` → `destroy()` → `destroy-method`
> **Prototype-scoped beans are NOT destroyed by the container** — you must manage their lifecycle manually.

---

### Complete Execution Order Example

```java
@Component
public class LifecycleDemoBean
        implements BeanNameAware, InitializingBean, DisposableBean {

    public LifecycleDemoBean() {
        System.out.println("1. Constructor called");
    }

    @Autowired
    public void setDependency(SomeDep dep) {
        System.out.println("2. Dependency injected");
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("3. BeanNameAware: " + name);
    }

    // BeanPostProcessor.postProcessBeforeInitialization() runs here (external)

    @PostConstruct
    public void postConstruct() {
        System.out.println("4. @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("5. afterPropertiesSet()");
    }

    // BeanPostProcessor.postProcessAfterInitialization() runs here (external)
    // Bean is now in use...

    @PreDestroy
    public void preDestroy() {
        System.out.println("6. @PreDestroy");
    }

    @Override
    public void destroy() {
        System.out.println("7. DisposableBean.destroy()");
    }
}
```

**Output:**
```
1. Constructor called
2. Dependency injected
3. BeanNameAware: lifecycleDemoBean
4. @PostConstruct
5. afterPropertiesSet()
--- bean in use ---
6. @PreDestroy
7. DisposableBean.destroy()
```

---

## Lifecycle Diagram

```
╔══════════════════════════════════════════════════════════════════╗
║                    SPRING IoC CONTAINER STARTS                   ║
╚══════════════════════════════════════════════════════════════════╝
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  PHASE 1: BEAN DEFINITION LOADING                                │
│                                                                  │
│   @ComponentScan ──┐                                             │
│   @Configuration ──┼──► BeanDefinitionRegistry                  │
│   XML Config    ───┘    (stores metadata for all beans)          │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  PHASE 2: BEAN INSTANTIATION                                     │
│                                                                  │
│   Spring picks one of:                                           │
│   ┌─────────────────────┐  ┌──────────────────┐                 │
│   │ Constructor Injection│  │  Factory Method  │                 │
│   └─────────────────────┘  └──────────────────┘                 │
│          └──────────────────────┘                                │
│                      │                                           │
│              Object is created                                   │
│            (raw, no deps yet)                                    │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  PHASE 3: DEPENDENCY INJECTION                                   │
│                                                                  │
│   @Autowired fields  ──┐                                         │
│   @Autowired setters ──┼──► Properties populated by container   │
│   @Value             ──┘                                         │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  PHASE 4: AWARE INTERFACES (in order)                            │
│                                                                  │
│   1. BeanNameAware         ──► setBeanName()                     │
│   2. BeanClassLoaderAware  ──► setBeanClassLoader()              │
│   3. BeanFactoryAware      ──► setBeanFactory()                  │
│   4. EnvironmentAware      ──► setEnvironment()                  │
│   5. ApplicationContextAware──► setApplicationContext()          │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  PHASE 5: BeanPostProcessor (BEFORE init)                        │
│                                                                  │
│   ALL registered BeanPostProcessors run:                         │
│   postProcessBeforeInitialization(bean, beanName)                │
│                                                                  │
│   Built-in examples:                                             │
│   ┌──────────────────────────────────────────────┐               │
│   │ AutowiredAnnotationBeanPostProcessor         │               │
│   │ CommonAnnotationBeanPostProcessor            │               │
│   └──────────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  PHASE 6: INITIALIZATION CALLBACKS (in order)                    │
│                                                                  │
│   Step A ──► @PostConstruct method                               │
│      │                                                           │
│      ▼                                                           │
│   Step B ──► InitializingBean.afterPropertiesSet()               │
│      │                                                           │
│      ▼                                                           │
│   Step C ──► @Bean(initMethod="...") / XML init-method           │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  PHASE 7: BeanPostProcessor (AFTER init)                         │
│                                                                  │
│   ALL registered BeanPostProcessors run:                         │
│   postProcessAfterInitialization(bean, beanName)                 │
│                                                                  │
│   ┌──────────────────────────────────────────────┐               │
│   │ AnnotationAwareAspectJAutoProxyCreator        │               │
│   │  └──► Wraps bean in AOP Proxy if needed       │               │
│   └──────────────────────────────────────────────┘               │
│                      │              │                            │
│               No AOP needed    AOP needed                        │
│                      │              │                            │
│               Original Bean    Proxy Bean                        │
│                      └──────────────┘                            │
│                             │                                    │
│                    Stored in Singleton Cache                      │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
╔══════════════════════════════════════════════════════════════════╗
║              BEAN IS READY — IN USE BY APPLICATION               ║
╚══════════════════════════════════════════════════════════════════╝
                               │
                               │  (application runs...)
                               │
                               ▼
╔══════════════════════════════════════════════════════════════════╗
║          CONTAINER SHUTDOWN (context.close() triggered)          ║
╚══════════════════════════════════════════════════════════════════╝
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  PHASE 8: DESTRUCTION CALLBACKS (in order)                       │
│                                                                  │
│   Step A ──► @PreDestroy method                                  │
│      │                                                           │
│      ▼                                                           │
│   Step B ──► DisposableBean.destroy()                            │
│      │                                                           │
│      ▼                                                           │
│   Step C ──► @Bean(destroyMethod="...") / XML destroy-method     │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
╔══════════════════════════════════════════════════════════════════╗
║                      BEAN DESTROYED                              ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## Singleton vs Prototype — Side-by-Side

```
         SINGLETON SCOPE                    PROTOTYPE SCOPE
         ───────────────                    ───────────────

   Container Starts                    Every getBean() call
         │                                     │
         ▼                                     ▼
   Instantiate (once)              Instantiate (new instance)
         │                                     │
         ▼                                     ▼
   Inject Dependencies             Inject Dependencies
         │                                     │
         ▼                                     ▼
   Aware Interfaces                Aware Interfaces
         │                                     │
         ▼                                     ▼
   BPP Before Init                 BPP Before Init
         │                                     │
         ▼                                     ▼
   Init Callbacks                  Init Callbacks
         │                                     │
         ▼                                     ▼
   BPP After Init                  BPP After Init
         │                                     │
         ▼                                     ▼
   Cached in Context               Handed to Caller
         │                         (container forgets it)
         ▼                                     │
   In Use...                       In Use...   │
         │                                     │
         ▼                                     ▼
   @PreDestroy        ◄──────   ✘ NOT called by container
   destroy()          ◄──────   ✘ NOT called by container
   destroy-method     ◄──────   ✘ Caller must manage manually
         │
         ▼
   Bean Destroyed
```

---

## BeanPostProcessor — How AOP Proxy Wrapping Works

```
  Original Bean                 BeanPostProcessor              Caller
       │                              │                           │
       │── postProcessAfterInit() ───►│                           │
       │                              │                           │
       │                      ┌───────┴──────────┐               │
       │                      │ Has @Transactional│               │
       │                      │ or @Aspect? (Y/N) │               │
       │                      └───────┬──────────┘               │
       │                           NO │      YES                  │
       │                              │       │                   │
       │◄─────── return original ─────┘       │                   │
       │                                      ▼                   │
       │                              Create Dynamic Proxy        │
       │                              wrapping Original Bean      │
       │                                      │                   │
       │                                      │──── stored ──────►│
       │                                                          │
       │                          Caller sees Proxy, not original │
```

---

## Key Differences: Singleton vs Prototype

| Aspect | Singleton | Prototype |
|---|---|---|
| Instances | One per container | New per request |
| Init callbacks | Called once | Called for each instance |
| Destroy callbacks | Called on shutdown | **NOT called** by container |
| Cached in context | Yes | No |

---

## Quick Reference Card

```
┌────┬──────────────────────────────────────┬────────────────────────┐
│ #  │ Lifecycle Step                        │ How to Hook In         │
├────┼──────────────────────────────────────┼────────────────────────┤
│ 1  │ Load Bean Definitions                 │ BeanFactoryPostProcessor│
│ 2  │ Instantiate                           │ Constructor / Factory  │
│ 3  │ Inject Dependencies                   │ @Autowired / @Value    │
│ 4  │ Aware Callbacks                       │ Implement XxxAware     │
│ 5  │ BPP Before Init                       │ BeanPostProcessor      │
│ 6a │ @PostConstruct                        │ @PostConstruct         │
│ 6b │ afterPropertiesSet()                  │ InitializingBean       │
│ 6c │ Custom init                           │ initMethod=""          │
│ 7  │ BPP After Init (AOP proxy here)       │ BeanPostProcessor      │
│ 8  │ BEAN IN USE                           │ —                      │
│ 9a │ @PreDestroy                           │ @PreDestroy            │
│ 9b │ destroy()                             │ DisposableBean         │
│ 9c │ Custom destroy                        │ destroyMethod=""       │
└────┴──────────────────────────────────────┴────────────────────────┘
```

---

## Summary: Recommended Practices

| Use Case | Recommended Approach |
|---|---|
| Initialization logic | `@PostConstruct` |
| Cleanup / resource release | `@PreDestroy` |
| Cross-cutting bean processing | `BeanPostProcessor` |
| Access to container internals | `Aware` interfaces (use sparingly) |
| Third-party class init/destroy | `@Bean(initMethod=..., destroyMethod=...)` |

---

> **Key Insight:** `BeanPostProcessor` is the most powerful extension point in the Spring lifecycle.
> It wraps every bean before and after initialization, which is the foundation for AOP,
> `@Transactional`, `@Async`, validation, and many other Spring features.
