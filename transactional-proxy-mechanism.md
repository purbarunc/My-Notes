# @Transactional Proxy Mechanism — Interview Answer

---

## The Core Idea

> "Spring's `@Transactional` works through **AOP proxies**. When you annotate a method or class with `@Transactional`, Spring doesn't modify your class directly — instead, it wraps your bean in a **proxy object** that intercepts calls and manages the transaction lifecycle around them."

---

## How It Works Step by Step

### 1. Bean Creation & Proxy Wrapping

When the Spring container initializes your bean, a `BeanPostProcessor` (specifically `InfrastructureAdvisorAutoProxyCreator`) detects `@Transactional` on the class or its methods. Instead of returning your plain object, Spring returns a **proxy** — either a:

- **JDK Dynamic Proxy** — if your bean implements an interface
- **CGLIB Proxy** — if it doesn't (subclasses your class at runtime)

### 2. Method Interception

When a caller invokes a `@Transactional` method on the injected bean, they're actually calling the **proxy**, not your class directly. The proxy delegates to `TransactionInterceptor`, which:

```
[Caller] → [Proxy] → [TransactionInterceptor] → [Your actual method]
```

### 3. Transaction Lifecycle

`TransactionInterceptor` calls `PlatformTransactionManager` to:

1. **Begin** a transaction (or join an existing one, depending on propagation)
2. **Invoke** your actual method
3. **Commit** on success
4. **Rollback** on unchecked exception (`RuntimeException` / `Error`) — or checked exceptions if explicitly configured

---

## The Famous "Self-Invocation" Pitfall

> "This proxy-based mechanism has one critical limitation interviewers love to ask about — **self-invocation**."

```java
public class OrderService {

    public void placeOrder() {
        // Calls processPayment() directly on `this` — bypasses the proxy!
        this.processPayment();
    }

    @Transactional
    public void processPayment() { ... }
}
```

Since `placeOrder()` calls `this.processPayment()`, it goes **directly to the real object**, skipping the proxy entirely. The `@Transactional` annotation on `processPayment()` is completely ignored.

---

## Fix Options for Self-Invocation (with Code Examples)

### Fix 1 — Self-Injection via `@Autowired` (Simplest)

Inject the Spring-managed proxy of the same bean into itself. The call now goes through the proxy, so `@Transactional` is honoured.

```java
@Service
public class OrderService {

    // Spring injects the proxy, not `this`
    @Autowired
    private OrderService self;

    public void placeOrder() {
        // Goes through the proxy — @Transactional is applied!
        self.processPayment();
    }

    @Transactional
    public void processPayment() {
        // transaction is active here
        System.out.println("Processing payment inside transaction...");
    }
}
```

> **Caveat:** Requires `spring.main.allow-circular-references=true` in Spring Boot 2.6+.  
> Add to `application.properties`:
> ```properties
> spring.main.allow-circular-references=true
> ```

---

### Fix 2 — Inject via `ApplicationContext` (Avoid Circular Reference Warning)

Fetch the proxy from the Spring context at runtime instead of declaring a circular `@Autowired`.

```java
@Service
public class OrderService implements ApplicationContextAware {

    private ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;
    }

    public void placeOrder() {
        // Fetch the proxy from the context at runtime
        OrderService proxy = context.getBean(OrderService.class);
        proxy.processPayment(); // goes through proxy — transaction applies
    }

    @Transactional
    public void processPayment() {
        System.out.println("Processing payment inside transaction...");
    }
}
```

> **Pros:** No circular dependency warning.  
> **Cons:** Couples your service to `ApplicationContext` (less testable).

---

### Fix 3 — Refactor into a Separate Bean (Best Practice)

Split the transactional method into its own dedicated service. This is the cleanest approach and aligns with Single Responsibility Principle.

```java
// Step 1: Extract the transactional logic into its own bean
@Service
public class PaymentService {

    @Transactional
    public void processPayment() {
        System.out.println("Processing payment inside transaction...");
    }
}

// Step 2: Inject that bean into OrderService
@Service
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    public void placeOrder() {
        // Cross-bean call → goes through PaymentService's proxy
        paymentService.processPayment(); // @Transactional is applied!
    }
}
```

> **Pros:** No hacks, clean design, fully testable, no circular dependency.  
> **Cons:** Requires structural refactoring.

---

### Fix 4 — Use `AopContext.currentProxy()` (Spring AOP Utility)

Spring exposes the current proxy via `AopContext`. You can use it to get the proxy reference inside the same class.

```java
@Service
@EnableAspectJAutoProxy(exposeProxy = true) // must expose proxy
public class OrderService {

    public void placeOrder() {
        // Get the current proxy reference from AopContext
        OrderService proxy = (OrderService) AopContext.currentProxy();
        proxy.processPayment(); // @Transactional is applied!
    }

    @Transactional
    public void processPayment() {
        System.out.println("Processing payment inside transaction...");
    }
}
```

Also enable `exposeProxy` in config or `application.properties`:

```java
// Java Config approach
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true)
public class AppConfig { }
```

> **Pros:** No separate bean needed, no circular reference issue.  
> **Cons:** Ties code to Spring AOP API — not ideal for pure unit testing.

---

### Fix 5 — AspectJ Weaving (Most Powerful, No Proxy Needed)

Switch from Spring's proxy-based AOP to **AspectJ weaving**, which instruments the bytecode of the actual class at compile-time or load-time. There is no proxy — `@Transactional` works even on `private` methods and self-calls.

**Step 1:** Add dependency in `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
</dependency>
```

**Step 2:** Enable AspectJ mode in config:

```java
@Configuration
@EnableTransactionManagement(mode = AdviceMode.ASPECTJ) // switch from PROXY to ASPECTJ
public class AppConfig { }
```

**Step 3:** Configure load-time weaving in `pom.xml` (for LTW):

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>-javaagent:path/to/spring-instrument.jar</argLine>
    </configuration>
</plugin>
```

**Step 4:** Your original code now works as-is:

```java
@Service
public class OrderService {

    public void placeOrder() {
        // With AspectJ weaving, this.processPayment() is intercepted
        // directly in the bytecode — no proxy needed!
        this.processPayment();
    }

    @Transactional
    public void processPayment() {
        System.out.println("Processing payment inside transaction...");
    }
}
```

> **Pros:** Most complete solution; works for private methods, self-calls, constructor injection.  
> **Cons:** More complex setup; requires build-time or agent-based weaving.

---

## Fix Options — Quick Comparison

| Fix | Complexity | Clean Code | Circular Ref Risk | Needs Refactor |
|---|---|---|---|---|
| Self-injection `@Autowired` | Low | Moderate | Yes (Spring Boot 2.6+) | No |
| `ApplicationContext.getBean()` | Low | Low | No | No |
| Separate Bean (Refactor) | Medium | High | No | Yes |
| `AopContext.currentProxy()` | Low | Moderate | No | No |
| AspectJ Weaving | High | High | No | No |

> **Interview answer:** "The cleanest fix is to refactor the transactional method into a separate Spring bean, so the call crosses the proxy boundary naturally. If refactoring isn't feasible, `AopContext.currentProxy()` or self-injection via `ApplicationContext` are pragmatic alternatives. For the most powerful solution, AspectJ weaving eliminates the proxy limitation entirely."

---

## Propagation & Isolation

> "The proxy also reads `propagation` and `isolation` attributes to decide how to manage the transaction context."

### Propagation Types

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing tx, or create new one |
| `REQUIRES_NEW` | Always suspend current tx, create new one |
| `SUPPORTS` | Join if exists, run non-transactionally otherwise |
| `MANDATORY` | Must have an existing tx, throws if none |
| `NOT_SUPPORTED` | Suspend current tx, run non-transactionally |
| `NEVER` | Must NOT have an existing tx, throws if one exists |
| `NESTED` | Run within a nested tx if one exists, else create new |

### Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible |
| `READ_COMMITTED` | Prevented | Possible | Possible |
| `REPEATABLE_READ` | Prevented | Prevented | Possible |
| `SERIALIZABLE` | Prevented | Prevented | Prevented |

---

## Proxy Types — JDK vs CGLIB

| | JDK Dynamic Proxy | CGLIB Proxy |
|---|---|---|
| **Requires** | Interface | No interface needed |
| **Mechanism** | `java.lang.reflect.Proxy` | Subclasses the target class at runtime |
| **Limitation** | Only proxies interface methods | Cannot proxy `final` classes or methods |
| **Spring Boot default** | Uses CGLIB by default since Spring Boot 2.x | — |

---

## Key Classes Involved

| Class | Role |
|---|---|
| `@EnableTransactionManagement` | Enables Spring's annotation-driven transaction management |
| `InfrastructureAdvisorAutoProxyCreator` | `BeanPostProcessor` that creates proxies |
| `TransactionInterceptor` | AOP advice that intercepts method calls |
| `PlatformTransactionManager` | Interface for transaction management (e.g., `JpaTransactionManager`) |
| `TransactionAttributeSource` | Reads `@Transactional` metadata from methods |

---

## Common Interview Gotchas

1. **`@Transactional` on `private` methods** — ignored; the proxy cannot intercept them
2. **`@Transactional` on `final` methods** — ignored with CGLIB (cannot be overridden)
3. **Self-invocation** — bypasses the proxy entirely (as shown above)
4. **Checked exceptions** — do NOT trigger rollback by default; must use `rollbackFor = Exception.class`
5. **`@Transactional` on the class vs method** — class-level applies to all public methods; method-level overrides it

```java
// Checked exception — will NOT rollback unless specified
@Transactional
public void riskyMethod() throws IOException {
    // rollback won't happen on IOException by default!
}

// Correct approach
@Transactional(rollbackFor = IOException.class)
public void riskyMethod() throws IOException { ... }
```

---

## One-liner Summary for Closing

> "In short: `@Transactional` is AOP in action — Spring generates a proxy at startup that wraps your bean, intercepts transactional method calls, and delegates transaction management to `PlatformTransactionManager`, giving you declarative transaction control without polluting business logic with infrastructure concerns."

---

---

# How It Works — Deep Dive: Behind the Scenes, Step by Step

---

## Overview: The Full Chain

```
Your Code (annotated)
    → Spring Container Startup
        → BeanPostProcessor scans beans
            → Detects @Transactional
                → Creates Proxy (JDK or CGLIB)
                    → Proxy registered in context (replaces real bean)
                        → Caller gets proxy (not real bean)
                            → Call intercepted by TransactionInterceptor
                                → PlatformTransactionManager begins tx
                                    → Your real method runs
                                        → Commit / Rollback
```

---

## Step 1 — You Write a `@Transactional` Class

```java
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        // imagine more DB operations here
    }
}
```

At this point, **nothing special has happened yet.** The annotation is just metadata sitting on the bytecode. Spring hasn't read it. No proxy exists. This is just a plain Java class.

---

## Step 2 — Spring Container Starts Up (`ApplicationContext.refresh()`)

The entry point for everything is `AbstractApplicationContext.refresh()`. This is the method that boots the entire Spring container.

```java
// Simplified from AbstractApplicationContext source
public void refresh() throws BeansException {
    // 1. Prepare context
    prepareRefresh();

    // 2. Create BeanFactory, load BeanDefinitions from XML/annotations
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // 3. Register BeanPostProcessors — THIS IS KEY
    registerBeanPostProcessors(beanFactory);

    // 4. Instantiate all non-lazy singleton beans
    finishBeanFactoryInitialization(beanFactory);
}
```

**`registerBeanPostProcessors`** is where Spring sets up the machinery that will later create proxies. `BeanPostProcessor` is a hook that runs **before and after** each bean is initialized.

---

## Step 3 — `@EnableTransactionManagement` Registers the Advisor

When you put `@EnableTransactionManagement` on a config class (or Spring Boot auto-configures it), it imports `ProxyTransactionManagementConfiguration`:

```java
@Configuration
@EnableTransactionManagement   // ← this triggers everything
public class AppConfig { }
```

**Behind the scenes**, `@EnableTransactionManagement` is a meta-annotation that imports:

```java
// What @EnableTransactionManagement actually does (simplified):
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {
    AdviceMode mode() default AdviceMode.PROXY;  // PROXY or ASPECTJ
    boolean proxyTargetClass() default false;
}
```

`TransactionManagementConfigurationSelector` then imports `ProxyTransactionManagementConfiguration`:

```java
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

    // Registers the Advisor — the "rule" that says which methods to intercept
    @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
            TransactionAttributeSource transactionAttributeSource,
            TransactionInterceptor transactionInterceptor) {

        BeanFactoryTransactionAttributeSourceAdvisor advisor =
            new BeanFactoryTransactionAttributeSourceAdvisor();

        // Tells advisor WHERE to look for @Transactional metadata
        advisor.setTransactionAttributeSource(transactionAttributeSource);

        // Tells advisor WHAT to do when intercepted (the actual tx logic)
        advisor.setAdvice(transactionInterceptor);

        return advisor;
    }

    // Reads @Transactional annotations from methods/classes
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionAttributeSource transactionAttributeSource() {
        return new AnnotationTransactionAttributeSource();
    }

    // The interceptor that manages the actual transaction
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionInterceptor transactionInterceptor(
            TransactionAttributeSource transactionAttributeSource) {

        TransactionInterceptor interceptor = new TransactionInterceptor();
        interceptor.setTransactionAttributeSource(transactionAttributeSource);
        return interceptor;
    }
}
```

Three key beans are now registered:

| Bean | Role |
|---|---|
| `AnnotationTransactionAttributeSource` | Reads `@Transactional` metadata from methods |
| `TransactionInterceptor` | The AOP advice — does the actual begin/commit/rollback |
| `BeanFactoryTransactionAttributeSourceAdvisor` | The Advisor — pairs the pointcut (which methods?) with the advice (what to do?) |

---

## Step 4 — `BeanPostProcessor` Scans Every Bean for `@Transactional`

`AbstractAutoProxyCreator` is the parent class of `InfrastructureAdvisorAutoProxyCreator`. It implements `BeanPostProcessor`. After every bean is initialized, Spring calls:

```java
// Called AFTER your bean is fully initialized
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        // Check if this bean should be wrapped in a proxy
        return wrapIfNecessary(bean, beanName, cacheKey);
    }
    return bean;
}
```

Inside `wrapIfNecessary`:

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {

    // Step 1: Find all Advisors that apply to this bean
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(
        bean.getClass(), beanName, null
    );

    // Step 2: If any advisor matched (i.e., @Transactional was found)
    if (specificInterceptors != DO_NOT_PROXY) {

        this.advisedBeans.put(cacheKey, Boolean.TRUE);

        // Step 3: Create the proxy!
        Object proxy = createProxy(
            bean.getClass(),       // your OrderService.class
            beanName,
            specificInterceptors,  // [TransactionInterceptor]
            new SingletonTargetSource(bean) // wraps your real instance
        );

        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;  // ← Spring now registers THIS in the context, not the real bean
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean; // No @Transactional found — return plain bean
}
```

`getAdvicesAndAdvisorsForBean` internally uses the `BeanFactoryTransactionAttributeSourceAdvisor` pointcut to check: *"Does this class or any of its public methods have `@Transactional`?"*

```java
// Inside AnnotationTransactionAttributeSource (simplified)
public TransactionAttribute getTransactionAttribute(Method method, Class<?> targetClass) {

    // Check method-level @Transactional first
    TransactionAttribute txAttr = findTransactionAttribute(method);
    if (txAttr != null) return txAttr;

    // Then class-level @Transactional
    txAttr = findTransactionAttribute(targetClass);
    if (txAttr != null) return txAttr;

    // Check the interface method if present
    Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);
    // ... repeat checks on specificMethod

    return null; // No @Transactional found
}
```

---

## Step 5 — Proxy Creation: JDK vs CGLIB Decision

Inside `createProxy`, Spring decides which proxy type to use:

```java
// Inside DefaultAopProxyFactory
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {

    // Use CGLIB if:
    // - proxyTargetClass = true (forced)
    // - OR the target class doesn't implement any user-defined interfaces
    if (config.isProxyTargetClass()
            || targetClass.isInterface() == false
            || Proxy.isProxyClass(targetClass)) {

        return new ObjenesisCglibAopProxy(config); // CGLIB
    }

    // Otherwise use JDK Dynamic Proxy
    return new JdkDynamicAopProxy(config); // JDK
}
```

### JDK Dynamic Proxy (interface-based)

```java
// JDK proxy requires an interface
public interface OrderServiceInterface {
    void placeOrder(Order order);
}

// The JDK proxy is created like this under the hood:
Object proxy = Proxy.newProxyInstance(
    targetClass.getClassLoader(),
    new Class[]{ OrderServiceInterface.class }, // interfaces
    jdkDynamicAopProxy  // this IS the InvocationHandler
);
```

`JdkDynamicAopProxy` itself implements `InvocationHandler`:

```java
// Simplified JdkDynamicAopProxy
class JdkDynamicAopProxy implements InvocationHandler {

    private final AdvisedSupport advised; // holds interceptors + real target

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // Get the chain of interceptors for this method
        List<Object> chain = advised.getInterceptorsAndDynamicInterceptionAdvice(
            method, targetClass
        );

        if (chain.isEmpty()) {
            // No interceptors — call the real method directly
            return AopUtils.invokeJoinpointUsingReflection(target, method, args);
        } else {
            // Run through the interceptor chain
            MethodInvocation invocation = new ReflectiveMethodInvocation(
                proxy, target, method, args, targetClass, chain
            );
            return invocation.proceed(); // ← this triggers TransactionInterceptor
        }
    }
}
```

### CGLIB Proxy (subclass-based)

CGLIB generates a **subclass** of your bean at runtime. The generated class looks something like this (conceptually):

```java
// What CGLIB generates at runtime (conceptual — not actual source):
public class OrderService$$EnhancerBySpringCGLIB$$a1b2c3d4 extends OrderService {

    private MethodInterceptor[] callbacks; // holds Spring's interceptor chain

    @Override
    public void placeOrder(Order order) {
        // Don't call super.placeOrder() directly
        // Instead, run through the Spring AOP interceptor chain
        MethodProxy.invoke(this, callbacks, new Object[]{ order });
        // callbacks[0] is DynamicAdvisedInterceptor → leads to TransactionInterceptor
    }
}
```

Internally CGLIB uses `DynamicAdvisedInterceptor`:

```java
// Inside CglibAopProxy
private static class DynamicAdvisedInterceptor implements MethodInterceptor {

    @Override
    public Object intercept(Object proxy, Method method,
                            Object[] args, MethodProxy methodProxy) throws Throwable {

        // Get interceptor chain for this method
        List<Object> chain = advised.getInterceptorsAndDynamicInterceptionAdvice(
            method, targetClass
        );

        if (chain.isEmpty()) {
            // No AOP needed — invoke real method via CGLIB MethodProxy (faster than reflection)
            return methodProxy.invoke(target, args);
        }

        CglibMethodInvocation invocation = new CglibMethodInvocation(
            proxy, target, method, args, targetClass, chain, methodProxy
        );
        return invocation.proceed(); // ← triggers TransactionInterceptor
    }
}
```

---

## Step 6 — The Proxy Lives in the Context (Your Bean is Hidden)

After `wrapIfNecessary` returns, Spring stores the **proxy** in the `BeanFactory` cache, not your original object. So when anyone does `@Autowired OrderService orderService`, they get the proxy.

```java
// Conceptually what the BeanFactory holds:
Map<String, Object> singletonObjects = {
    "orderService" → OrderService$$EnhancerBySpringCGLIB$$a1b2c3d4  // PROXY
    // The real OrderService instance is buried inside the proxy's target
};
```

---

## Step 7 — Method Call Intercepted: `TransactionInterceptor.invoke()`

This is the heart of `@Transactional`. When you call `orderService.placeOrder(order)`:

```
[Your Code]
   ↓ calls placeOrder() on the proxy
[CGLIB / JDK Proxy]
   ↓ builds interceptor chain, calls proceed()
[ReflectiveMethodInvocation.proceed()]
   ↓ calls next interceptor in chain
[TransactionInterceptor.invoke()]      ← WE ARE HERE
   ↓ delegates to
[TransactionAspectSupport.invokeWithinTransaction()]
```

Here is the actual `TransactionInterceptor.invoke()` simplified:

```java
public class TransactionInterceptor extends TransactionAspectSupport
                                     implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // Get the actual target class (not proxy class)
        Class<?> targetClass = (invocation.getThis() != null ?
            AopUtils.getTargetClass(invocation.getThis()) : null);

        // Delegate all transaction logic to parent class
        return invokeWithinTransaction(
            invocation.getMethod(),
            targetClass,
            invocation::proceed  // ← this calls your real method when invoked
        );
    }
}
```

---

## Step 8 — `invokeWithinTransaction()`: The Full Transaction Lifecycle

This is the deepest and most important method. Here is a heavily annotated simplified version:

```java
// Inside TransactionAspectSupport
protected Object invokeWithinTransaction(Method method,
                                          Class<?> targetClass,
                                          InvocationCallback invocation) throws Throwable {

    // ── 1. READ @Transactional METADATA ──────────────────────────────────────
    TransactionAttributeSource tas = getTransactionAttributeSource();

    // Reads: propagation, isolation, timeout, readOnly, rollbackFor etc.
    final TransactionAttribute txAttr = tas.getTransactionAttribute(method, targetClass);

    // ── 2. RESOLVE WHICH PlatformTransactionManager TO USE ───────────────────
    final TransactionManager tm = determineTransactionManager(txAttr);
    // e.g., JpaTransactionManager, DataSourceTransactionManager, etc.

    final String joinpointIdentification = methodIdentification(method, targetClass);

    // ── 3. STANDARD PROXY-BASED TRANSACTION (AdviceMode.PROXY) ──────────────
    if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {

        // ── 3a. BEGIN / JOIN TRANSACTION ─────────────────────────────────────
        // This respects the Propagation setting (REQUIRED, REQUIRES_NEW, etc.)
        TransactionInfo txInfo = createTransactionIfNecessary(
            (PlatformTransactionManager) tm, txAttr, joinpointIdentification
        );

        Object retVal;
        try {
            // ── 3b. INVOKE YOUR REAL METHOD ───────────────────────────────────
            // This is where your actual business code runs
            retVal = invocation.proceedWithInvocation();

        } catch (Throwable ex) {

            // ── 3c. HANDLE EXCEPTION → ROLLBACK OR COMMIT? ───────────────────
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;

        } finally {
            // ── 3d. CLEAN UP ThreadLocal transaction state ────────────────────
            cleanupTransactionInfo(txInfo);
        }

        // ── 3e. COMMIT ────────────────────────────────────────────────────────
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }
}
```

---

## Step 9 — `createTransactionIfNecessary()`: Propagation Logic

This is where **propagation** decisions are made:

```java
protected TransactionInfo createTransactionIfNecessary(
        PlatformTransactionManager tm,
        TransactionAttribute txAttr,
        String joinpointIdentification) {

    // Get current transaction status — checks ThreadLocal for existing tx
    TransactionStatus status = tm.getTransaction(txAttr);
    // ↑ This is where REQUIRED vs REQUIRES_NEW vs SUPPORTS etc. is resolved

    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

Inside `PlatformTransactionManager.getTransaction()` (e.g., `JpaTransactionManager`):

```java
// Simplified AbstractPlatformTransactionManager.getTransaction()
public final TransactionStatus getTransaction(TransactionDefinition definition) {

    // Check if there's already a transaction bound to the current thread
    Object transaction = doGetTransaction(); // checks ThreadLocal

    if (isExistingTransaction(transaction)) {
        // An existing transaction is active — handle based on propagation:
        return handleExistingTransaction(definition, transaction);
        // REQUIRED      → join it (do nothing, reuse)
        // REQUIRES_NEW  → suspend current, start fresh
        // NOT_SUPPORTED → suspend current, run without tx
        // MANDATORY     → join it (ok)
        // NEVER         → throw exception
        // NESTED        → create savepoint within current tx
    }

    // No existing transaction:
    if (definition.getPropagationBehavior() == PROPAGATION_REQUIRED ||
        definition.getPropagationBehavior() == PROPAGATION_REQUIRES_NEW) {

        SuspendedResourcesHolder suspendedResources = suspend(null);
        try {
            // ← Actually begins a DB transaction here
            return startTransaction(definition, transaction, suspendedResources);
        }
    }

    // PROPAGATION_SUPPORTS / NOT_SUPPORTED / NEVER with no existing tx:
    return prepareTransactionStatus(definition, null, true, false, null);
}
```

### How Transaction State is Stored (ThreadLocal)

The active transaction is bound to the **current thread** via `TransactionSynchronizationManager`:

```java
// Spring binds the Connection/EntityManager to current thread
public abstract class TransactionSynchronizationManager {

    // Key = DataSource or EntityManagerFactory, Value = Connection/EntityManager holder
    private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");

    private static final ThreadLocal<Boolean> actualTransactionActive =
        new NamedThreadLocal<>("Actual transaction active");

    private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
        new NamedThreadLocal<>("Current transaction isolation level");

    // When a transaction begins:
    public static void bindResource(Object key, Object value) {
        Map<Object, Object> map = resources.get();
        if (map == null) {
            map = new HashMap<>();
            resources.set(map);
        }
        map.put(key, value); // e.g., DataSource → ConnectionHolder
    }
}
```

> This is why transactions are **thread-scoped** and why you cannot share them across threads.

---

## Step 10 — `completeTransactionAfterThrowing()`: Rollback Logic

```java
protected void completeTransactionAfterThrowing(TransactionInfo txInfo, Throwable ex) {

    if (txInfo.transactionAttribute != null && txInfo.hasTransaction()) {

        // Check: should this exception trigger a rollback?
        if (txInfo.transactionAttribute.rollbackOn(ex)) {
            // YES — rollback
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
        } else {
            // NO  — commit even though an exception was thrown
            // (This is why checked exceptions DON'T rollback by default!)
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
        }
    }
}
```

The `rollbackOn()` check:

```java
// Default rollback rule in RuleBasedTransactionAttribute:
public boolean rollbackOn(Throwable ex) {
    // Default: only rollback on RuntimeException and Error
    // Checked exceptions (IOException, SQLException, etc.) → COMMIT by default
    return (ex instanceof RuntimeException || ex instanceof Error);
}

// When you write @Transactional(rollbackFor = IOException.class):
// rollbackOn() additionally checks your configured rollbackFor list
```

---

## Step 11 — Commit Path

```java
protected void commitTransactionAfterReturning(TransactionInfo txInfo) {
    if (txInfo != null && txInfo.hasTransaction()) {
        // Calls PlatformTransactionManager.commit()
        txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
    }
}
```

Inside `JpaTransactionManager.doCommit()`:

```java
@Override
protected void doCommit(DefaultTransactionStatus status) {
    JpaTransactionObject txObject = (JpaTransactionObject) status.getTransaction();
    EntityManager em = txObject.getEntityManagerHolder().getEntityManager();
    try {
        EntityTransaction tx = em.getTransaction();
        tx.commit(); // ← actual JDBC/JPA commit
    } catch (RollbackException ex) {
        throw new TransactionSystemException("Could not commit JPA transaction", ex);
    }
}
```

---

## Full Visual Flow (All Steps Together)

```
Spring Boot starts
│
├─ @EnableTransactionManagement detected
│   └─ Registers:
│       ├─ AnnotationTransactionAttributeSource
│       ├─ TransactionInterceptor
│       └─ BeanFactoryTransactionAttributeSourceAdvisor
│
├─ OrderService bean instantiated
│   └─ BeanPostProcessor.postProcessAfterInitialization() fires
│       └─ wrapIfNecessary()
│           ├─ Checks: does OrderService have @Transactional? → YES
│           ├─ Decides: JDK Proxy or CGLIB? → CGLIB (no interface / Boot 2.x+)
│           └─ Returns CGLIB proxy, registered in BeanFactory
│
└─ Application runs
    │
    └─ @Autowired OrderService orderService → gets PROXY
        │
        └─ orderService.placeOrder(order) called
            │
            └─ CGLIB DynamicAdvisedInterceptor.intercept()
                │
                └─ ReflectiveMethodInvocation.proceed()
                    │
                    └─ TransactionInterceptor.invoke()
                        │
                        └─ invokeWithinTransaction()
                            │
                            ├─ Read @Transactional metadata (propagation=REQUIRED, etc.)
                            ├─ Resolve PlatformTransactionManager (JpaTransactionManager)
                            ├─ createTransactionIfNecessary()
                            │   ├─ Check ThreadLocal for existing tx → none found
                            │   └─ Begin new tx → bind Connection to ThreadLocal
                            │
                            ├─ invocation.proceedWithInvocation()
                            │   └─ Your real OrderService.placeOrder() runs
                            │       └─ orderRepository.save(order)
                            │           └─ Uses same Connection from ThreadLocal
                            │
                            ├─ (on success) commitTransactionAfterReturning()
                            │   └─ JpaTransactionManager.commit()
                            │       └─ EntityTransaction.commit() → JDBC commit
                            │
                            └─ (on RuntimeException) completeTransactionAfterThrowing()
                                └─ rollbackOn(ex) == true?
                                    ├─ YES → tm.rollback()
                                    └─ NO  → tm.commit() (checked ex default)
```

---

## Key Insight: Why Self-Invocation Fails (Visualized)

```java
// What happens with self-invocation:

orderService.placeOrder();  // ← goes through PROXY ✓
    │
    └─ Real OrderService.placeOrder() runs
        │
        └─ this.processPayment()  // ← `this` is the REAL object, NOT the proxy
            │
            └─ Real OrderService.processPayment() runs DIRECTLY
                │
                └─ TransactionInterceptor is NEVER called
                   @Transactional is IGNORED ✗
```

The proxy only intercepts calls that come **from outside the object**. Once you're inside the real object, `this` points to the raw instance — the proxy is completely bypassed.

---

## Key Classes — What Each One Actually Does

| Class | Package | What It Does |
|---|---|---|
| `AbstractApplicationContext.refresh()` | `context.support` | Boots entire container, triggers proxy creation |
| `AbstractAutoProxyCreator` | `aop.framework.autoproxy` | `BeanPostProcessor` — decides whether to wrap a bean in a proxy |
| `InfrastructureAdvisorAutoProxyCreator` | `aop.framework.autoproxy` | Subclass specifically for infrastructure advisors like `@Transactional` |
| `BeanFactoryTransactionAttributeSourceAdvisor` | `transaction.interceptor` | The Advisor — combines pointcut (where?) + advice (what?) |
| `AnnotationTransactionAttributeSource` | `transaction.interceptor` | Reads `@Transactional` metadata from method/class |
| `TransactionInterceptor` | `transaction.interceptor` | The AOP Advice — implements `MethodInterceptor`, entry point for tx logic |
| `TransactionAspectSupport` | `transaction.interceptor` | Parent of `TransactionInterceptor` — contains `invokeWithinTransaction()` |
| `PlatformTransactionManager` | `transaction` | Interface; `JpaTransactionManager` etc. implement actual begin/commit/rollback |
| `TransactionSynchronizationManager` | `transaction.support` | Stores active tx (Connection/EntityManager) in `ThreadLocal` |
| `JdkDynamicAopProxy` | `aop.framework` | Creates JDK proxy; implements `InvocationHandler` |
| `ObjenesisCglibAopProxy` | `aop.framework` | Creates CGLIB subclass proxy |
| `ReflectiveMethodInvocation` | `aop.framework` | Walks the interceptor chain; calls `proceed()` on each |

---

---

# "I Never Wrote `@EnableTransactionManagement` — So How Does It Still Work?"

> **Short answer:** Spring Boot auto-configures it for you via `TransactionAutoConfiguration`. You never need to write `@EnableTransactionManagement` in a Spring Boot app — Boot does it silently based on what's on your classpath.

---

## The Entry Point: `@SpringBootApplication`

Everything starts from the annotation you put on your main class:

```java
@SpringBootApplication   // ← this one annotation triggers everything
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

`@SpringBootApplication` is itself a composed annotation:

```java
// What @SpringBootApplication actually is:
@SpringBootConfiguration      // ← same as @Configuration
@EnableAutoConfiguration      // ← THIS triggers auto-configuration magic
@ComponentScan                // ← scans your packages for @Component, @Service etc.
public @interface SpringBootApplication { }
```

`@EnableAutoConfiguration` is the key — it tells Spring Boot to load all auto-configuration classes automatically.

---

## How Auto-Configuration Is Loaded

Spring Boot maintains a list of all auto-configuration classes. In **Spring Boot 2.x** it's in:

```
META-INF/spring.factories
```

In **Spring Boot 3.x** it moved to:

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

When you look inside the `spring-boot-autoconfigure` jar, you'll find `TransactionAutoConfiguration` listed there:

```properties
# spring.factories (Spring Boot 2.x) — inside spring-boot-autoconfigure.jar
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  ...
  org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration,\
  ...
```

```text
# AutoConfiguration.imports (Spring Boot 3.x) — inside spring-boot-autoconfigure.jar
...
org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration
...
```

Spring Boot reads this file at startup and loads every class listed — **automatically**, without you writing a single line.

---

## Step-by-Step: What Happens Under the Hood

### Stage 1 — `@EnableAutoConfiguration` Activates `AutoConfigurationImportSelector`

```java
// @EnableAutoConfiguration imports this selector
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration { }
```

`AutoConfigurationImportSelector` reads the `.factories` / `.imports` file and returns all the auto-config class names to Spring's `BeanDefinitionRegistry`. They're all registered as candidate beans — but **conditionally** (more on that below).

---

### Stage 2 — `TransactionAutoConfiguration` is Loaded

Here is the actual simplified source of `TransactionAutoConfiguration`:

```java
@AutoConfiguration
@ConditionalOnClass(PlatformTransactionManager.class)  // ← only if tx manager is on classpath
@EnableConfigurationProperties(TransactionProperties.class)
public class TransactionAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public TransactionManagerCustomizersBeans transactionManagerCustomizers(...) { ... }

    // ─────────────────────────────────────────────────────────────────────────
    // THIS is the inner class that actually enables @Transactional support:
    // ─────────────────────────────────────────────────────────────────────────
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnBean(TransactionManager.class)         // ← only if a tx manager bean EXISTS
    @ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)
    public static class EnableTransactionManagementConfiguration {

        @Configuration(proxyBeanMethods = false)
        @EnableTransactionManagement(proxyTargetClass = false)  // ← HERE IT IS
        @ConditionalOnProperty(
            prefix = "spring.aop",
            name = "proxy-target-class",
            havingValue = "false"
        )
        public static class JdkDynamicAutoProxyConfiguration { }

        @Configuration(proxyBeanMethods = false)
        @EnableTransactionManagement(proxyTargetClass = true)   // ← OR THIS (CGLIB)
        @ConditionalOnProperty(
            prefix = "spring.aop",
            name = "proxy-target-class",
            matchIfMissing = true  // ← TRUE by default — so CGLIB is the default
        )
        public static class CglibAutoProxyConfiguration { }
    }
}
```

**Boot auto-applies `@EnableTransactionManagement` for you** inside `EnableTransactionManagementConfiguration`. It even decides JDK vs CGLIB based on your `application.properties`.

---

### Stage 3 — `@Conditional` Guards Decide Whether to Activate

Auto-configuration doesn't blindly activate everything. Each class has `@Conditional` guards:

```
@ConditionalOnClass(PlatformTransactionManager.class)
```
> "Only load this auto-config if `PlatformTransactionManager` is present on the classpath."
> — satisfied as soon as you add `spring-tx` (included by `spring-boot-starter-data-jpa` etc.)

```
@ConditionalOnBean(TransactionManager.class)
```
> "Only enable `@EnableTransactionManagement` if a `TransactionManager` bean has already been registered."
> — satisfied when `JpaTransactionManager` or `DataSourceTransactionManager` is auto-configured.

```
@ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)
```
> "Only activate this if YOU haven't already provided your own `@EnableTransactionManagement`."
> — prevents double-registration if you explicitly add it yourself.

---

### Stage 4 — Who Creates the `TransactionManager` Bean?

You never create a `JpaTransactionManager` yourself either — Boot auto-configures that too, via `JpaBaseConfiguration`:

```java
// Inside HibernateJpaConfiguration (extends JpaBaseConfiguration)
@Bean
@ConditionalOnMissingBean(TransactionManager.class)
public PlatformTransactionManager transactionManager(
        ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {

    JpaTransactionManager transactionManager =
        new JpaTransactionManager();  // ← created automatically

    transactionManagerCustomizers.ifAvailable(
        customizers -> customizers.customize(transactionManager)
    );
    return transactionManager;
}
```

This bean satisfies the `@ConditionalOnBean(TransactionManager.class)` check in `TransactionAutoConfiguration`, which then activates `@EnableTransactionManagement`.

---

## The Full Auto-Configuration Chain

```
@SpringBootApplication
│
└─ @EnableAutoConfiguration
    │
    └─ AutoConfigurationImportSelector reads spring.factories / AutoConfiguration.imports
        │
        ├─ HibernateJpaAutoConfiguration  (because spring-boot-starter-data-jpa is on classpath)
        │   └─ JpaBaseConfiguration
        │       └─ @Bean JpaTransactionManager  ← PlatformTransactionManager bean is born
        │
        └─ TransactionAutoConfiguration
            │
            ├─ @ConditionalOnClass(PlatformTransactionManager.class) → PASS ✓
            │
            └─ EnableTransactionManagementConfiguration
                │
                ├─ @ConditionalOnBean(TransactionManager.class) → PASS ✓ (JpaTransactionManager exists)
                ├─ @ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class) → PASS ✓
                │
                └─ CglibAutoProxyConfiguration  (spring.aop.proxy-target-class=true by default)
                    │
                    └─ @EnableTransactionManagement(proxyTargetClass = true)
                        │
                        └─ Imports ProxyTransactionManagementConfiguration
                            └─ Registers: AnnotationTransactionAttributeSource
                                          TransactionInterceptor
                                          BeanFactoryTransactionAttributeSourceAdvisor
                                          ↑
                                          (Everything from Step 3 of the deep dive above kicks in)
```

---

## What Happens if You Add `@EnableTransactionManagement` Yourself?

Nothing breaks. The `@ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)` guard detects that you've already provided configuration and **skips** Boot's auto-config for it. Your explicit annotation takes precedence.

```java
// This is fine — Boot will back off and let yours win
@Configuration
@EnableTransactionManagement
public class MyConfig { }
```

---

## What Triggers the Whole Chain: Classpath Detection Summary

| You add to `pom.xml` / `build.gradle` | Boot auto-configures |
|---|---|
| `spring-boot-starter-data-jpa` | `JpaTransactionManager` + `@EnableTransactionManagement` |
| `spring-boot-starter-jdbc` | `DataSourceTransactionManager` + `@EnableTransactionManagement` |
| `spring-boot-starter-data-r2dbc` | `R2dbcTransactionManager` + reactive tx support |
| Any custom `PlatformTransactionManager` `@Bean` | `@EnableTransactionManagement` still auto-applied |

> You only need `@EnableTransactionManagement` in a **plain Spring (non-Boot)** application, or when you want to override Boot's defaults (e.g., force JDK proxies instead of CGLIB).

---

## One-liner Summary

> Spring Boot's `TransactionAutoConfiguration` detects a `PlatformTransactionManager` bean on the context and automatically applies `@EnableTransactionManagement` for you — the entire proxy/interceptor machinery from `ProxyTransactionManagementConfiguration` activates without you writing a single configuration line.
