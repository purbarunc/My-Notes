# SOLID Principles — Complete Guide

---

## Table of Contents

1. [What is SOLID?](#what-is-solid)
2. [Origin — Who Termed SOLID?](#origin--who-termed-solid)
3. [S — Single Responsibility Principle (SRP)](#s--single-responsibility-principle-srp)
4. [O — Open/Closed Principle (OCP)](#o--openclosed-principle-ocp)
5. [L — Liskov Substitution Principle (LSP)](#l--liskov-substitution-principle-lsp)
6. [I — Interface Segregation Principle (ISP)](#i--interface-segregation-principle-isp)
7. [D — Dependency Inversion Principle (DIP)](#d--dependency-inversion-principle-dip)
8. [Quick Summary Table](#quick-summary-table)
9. [Real-World Analogy](#real-world-analogy)
10. [How OOP Implements SOLID](#how-oop-implements-solid)
11. [How Spring Framework Implements SOLID](#how-spring-framework-implements-solid)
12. [Complete Mapping: OOP + Spring → SOLID](#complete-mapping-oop--spring--solid)
13. [The Big Picture](#the-big-picture)

---

## What is SOLID?

**SOLID** is an acronym for 5 design principles that make software easier to maintain, extend, and understand.

| Letter | Principle | Core Idea |
|--------|-----------|-----------|
| **S** | Single Responsibility Principle | One class, one job |
| **O** | Open/Closed Principle | Extend, don't modify |
| **L** | Liskov Substitution Principle | Subclasses honor parent's contract |
| **I** | Interface Segregation Principle | Small focused interfaces |
| **D** | Dependency Inversion Principle | Depend on abstractions |

---

## Origin — Who Termed SOLID?

### Robert C. Martin — "Uncle Bob"

The five principles were identified and introduced by **Robert C. Martin**, a software engineer and author, popularly known as **"Uncle Bob"**.

He first introduced these principles in his paper:
> **"Design Principles and Design Patterns"** — published around **2000**

He later expanded on them in his famous book:
> **"Agile Software Development, Principles, Patterns, and Practices"** (2002)

### The Acronym "SOLID" — Michael Feathers

Interestingly, **Robert C. Martin did NOT coin the acronym "SOLID"** himself.

The **SOLID acronym** was later coined by **Michael Feathers** (author of *"Working Effectively with Legacy Code"*), who noticed that the five principles Robert C. Martin described spelled out SOLID and popularized it in that form.

### Timeline at a Glance

| Year | Event |
|------|-------|
| ~2000 | Robert C. Martin publishes "Design Principles and Design Patterns" |
| 2002 | Uncle Bob's book *Agile Software Development* formalizes the principles |
| ~2004 | Michael Feathers coins the **SOLID** acronym |

### About Robert C. Martin (Uncle Bob)

- Co-author of the **Agile Manifesto** (2001)
- Wrote influential books: *Clean Code*, *Clean Architecture*, *The Clean Coder*
- Strong advocate of software craftsmanship and professional ethics in software engineering
- His blog: **blog.cleancoder.com**

> In short — **Uncle Bob gave us the principles, Michael Feathers gave us the memorable acronym.**

---

## S — Single Responsibility Principle (SRP)

> **"A class should have only ONE reason to change."**

Every class should do **one thing and one thing only**.

### Bad Example (Violates SRP)

```java
// This class does TOO MANY things
public class Employee {
    private String name;
    private double salary;

    public void calculateSalary() {
        // calculates salary logic
    }

    public void saveToDatabase() {
        // DB logic — Employee shouldn't know about DB!
    }

    public void generatePayslipPDF() {
        // PDF logic — Employee shouldn't know about PDFs!
    }
}
```

### Good Example (Follows SRP)

```java
// Each class has ONE job
public class Employee {
    private String name;
    private double salary;

    public double getSalary() { return salary; }
    public String getName() { return name; }
}

public class SalaryCalculator {
    public double calculate(Employee emp) {
        return emp.getSalary() * 1.1; // apply bonus, etc.
    }
}

public class EmployeeRepository {
    public void save(Employee emp) {
        // only handles DB persistence
    }
}

public class PayslipGenerator {
    public void generate(Employee emp) {
        // only handles PDF generation
    }
}
```

**Why it matters:** If the PDF format changes, you only touch `PayslipGenerator`. You don't risk breaking salary logic.

---

## O — Open/Closed Principle (OCP)

> **"A class should be OPEN for extension but CLOSED for modification."**

You should be able to add new behavior **without changing existing code**.

### Bad Example (Violates OCP)

```java
public class DiscountService {
    public double getDiscount(String customerType) {
        if (customerType.equals("REGULAR")) return 0.05;
        else if (customerType.equals("PREMIUM")) return 0.15;
        // Every new customer type = modify this class = risky!
        return 0;
    }
}
```

### Good Example (Follows OCP)

```java
// Define an abstraction
public interface DiscountStrategy {
    double getDiscount();
}

// Extend by adding new classes, never touching old ones
public class RegularCustomerDiscount implements DiscountStrategy {
    public double getDiscount() { return 0.05; }
}

public class PremiumCustomerDiscount implements DiscountStrategy {
    public double getDiscount() { return 0.15; }
}

// New requirement: add VIP customer — NO modification needed
public class VIPCustomerDiscount implements DiscountStrategy {
    public double getDiscount() { return 0.30; }
}

public class DiscountService {
    public double getDiscount(DiscountStrategy strategy) {
        return strategy.getDiscount(); // closed for modification
    }
}
```

**Usage:**

```java
DiscountService service = new DiscountService();
double d = service.getDiscount(new VIPCustomerDiscount()); // 0.30
```

**Why it matters:** Adding a new customer type is just adding a new class — no risk of breaking existing discount logic.

---

## L — Liskov Substitution Principle (LSP)

> **"Subclasses should be substitutable for their parent class WITHOUT breaking the program."**

If `Bird` is the parent, and you substitute it with `Penguin` (a subclass), the program should still work correctly.

### Bad Example (Violates LSP)

```java
public class Bird {
    public void fly() {
        System.out.println("Flying...");
    }
}

public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins can't fly!");
        // This BREAKS the contract — substituting Penguin for Bird crashes the program
    }
}

public class Main {
    public static void makeBirdFly(Bird bird) {
        bird.fly(); // Crashes if bird is a Penguin!
    }
}
```

### Good Example (Follows LSP)

```java
// Split behaviors properly
public interface Bird {
    void eat();
}

public interface FlyingBird extends Bird {
    void fly();
}

public class Sparrow implements FlyingBird {
    public void eat() { System.out.println("Sparrow eating"); }
    public void fly() { System.out.println("Sparrow flying"); }
}

public class Penguin implements Bird {
    public void eat() { System.out.println("Penguin eating"); }
    // No fly() — doesn't violate any contract
}

public class Main {
    public static void makeFlyingBirdFly(FlyingBird bird) {
        bird.fly(); // Safe — only called on birds that CAN fly
    }
}
```

**Why it matters:** Any subclass can safely replace its parent without unexpected behavior.

---

## I — Interface Segregation Principle (ISP)

> **"A class should NOT be forced to implement interfaces it doesn't use."**

Prefer many **small, specific interfaces** over one large, fat interface.

### Bad Example (Violates ISP)

```java
// Fat interface — forces all implementors to define ALL methods
public interface Worker {
    void work();
    void eat();
    void sleep();
}

public class Robot implements Worker {
    public void work() { System.out.println("Robot working"); }
    public void eat() { /* Robots don't eat! Forced to implement anyway */ }
    public void sleep() { /* Robots don't sleep! Forced to implement anyway */ }
}
```

### Good Example (Follows ISP)

```java
// Split into focused interfaces
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

// Human uses all three
public class HumanWorker implements Workable, Eatable, Sleepable {
    public void work()  { System.out.println("Human working"); }
    public void eat()   { System.out.println("Human eating"); }
    public void sleep() { System.out.println("Human sleeping"); }
}

// Robot only uses what it needs
public class Robot implements Workable {
    public void work() { System.out.println("Robot working"); }
    // No forced empty methods!
}
```

**Why it matters:** Classes only implement what they actually need — no dead/empty method implementations.

---

## D — Dependency Inversion Principle (DIP)

> **"High-level modules should NOT depend on low-level modules. Both should depend on ABSTRACTIONS."**

Your business logic (high-level) should not be tightly coupled to implementation details (low-level).

### Bad Example (Violates DIP)

```java
// High-level class is tightly coupled to low-level class
public class MySQLDatabase {
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

public class OrderService {
    private MySQLDatabase database = new MySQLDatabase(); // Direct dependency!

    public void placeOrder(String order) {
        // If we switch to PostgreSQL, we MUST modify OrderService
        database.save(order);
    }
}
```

### Good Example (Follows DIP)

```java
// 1. Define an abstraction (interface)
public interface Database {
    void save(String data);
}

// 2. Low-level modules implement the abstraction
public class MySQLDatabase implements Database {
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

public class PostgreSQLDatabase implements Database {
    public void save(String data) {
        System.out.println("Saving to PostgreSQL: " + data);
    }
}

public class MongoDatabase implements Database {
    public void save(String data) {
        System.out.println("Saving to MongoDB: " + data);
    }
}

// 3. High-level module depends on abstraction, not implementation
public class OrderService {
    private Database database; // depends on interface, not concrete class

    // Inject dependency from outside (Dependency Injection)
    public OrderService(Database database) {
        this.database = database;
    }

    public void placeOrder(String order) {
        database.save(order); // works with ANY database!
    }
}
```

**Usage:**

```java
// Swap databases without touching OrderService at all
OrderService mysqlService    = new OrderService(new MySQLDatabase());
OrderService postgresService = new OrderService(new PostgreSQLDatabase());
OrderService mongoService    = new OrderService(new MongoDatabase());

mysqlService.placeOrder("Order #1");    // Saving to MySQL: Order #1
postgresService.placeOrder("Order #2"); // Saving to PostgreSQL: Order #2
```

**Why it matters:** Swap implementations freely — no ripple effect on business logic.

---

## Quick Summary Table

| Principle | One-Line Rule | Key Benefit |
|-----------|---------------|-------------|
| **S**RP | One class, one job | Easier to maintain & test |
| **O**CP | Extend, don't modify | Safe to add new features |
| **L**SP | Subclasses honor parent's contract | Reliable polymorphism |
| **I**SP | Small focused interfaces | No forced empty methods |
| **D**IP | Depend on abstractions | Loose coupling, easy swapping |

---

## Real-World Analogy

Think of building a **smart home**:

- **SRP** — Thermostat controls temperature only. Alarm system handles security only.
- **OCP** — Add a new smart device (e.g., smart lock) without rewiring the whole house.
- **LSP** — Any "light switch" (smart or dumb) works the same way in any socket.
- **ISP** — A basic bulb only needs an "on/off" interface, not a "dim/color/schedule" interface.
- **DIP** — Your home app talks to a `SmartDevice` interface, not directly to a specific brand's SDK.

Following SOLID makes your codebase **resilient to change** — new features are additions, not rewrites.

---

## How OOP Implements SOLID

OOP has 4 core pillars — **Encapsulation, Abstraction, Inheritance, Polymorphism** — and each one directly enables one or more SOLID principles.

---

### Encapsulation → Supports SRP

Encapsulation bundles data and behavior together, hiding internal details. This naturally enforces **one responsibility per class**.

```java
// Encapsulation keeps Employee data and behavior together,
// preventing other classes from mixing concerns into it
public class Employee {
    private String name;
    private double salary; // hidden — only Employee manages its own data

    public String getName() { return name; }
    public double getSalary() { return salary; }

    // Only Employee knows how to validate itself
    public boolean isValid() {
        return name != null && salary > 0;
    }
}
```

Encapsulation forces you to ask: *"Does this behavior belong HERE?"* — which is exactly what SRP asks.

---

### Abstraction → Supports OCP + DIP

Abstraction (via `abstract` classes and `interface`) lets you define **what** something does without caring **how**.

```java
// Abstraction: define WHAT a payment does, not HOW
public interface PaymentGateway {
    void processPayment(double amount);
    void refund(double amount);
}

// HOW is defined in implementations — extend freely without modifying above
public class StripeGateway implements PaymentGateway {
    public void processPayment(double amount) { /* Stripe logic */ }
    public void refund(double amount) { /* Stripe refund */ }
}

public class PayPalGateway implements PaymentGateway {
    public void processPayment(double amount) { /* PayPal logic */ }
    public void refund(double amount) { /* PayPal refund */ }
}

// High-level code depends on abstraction — DIP
public class CheckoutService {
    private PaymentGateway gateway; // depends on interface, not class

    public CheckoutService(PaymentGateway gateway) {
        this.gateway = gateway;
    }

    public void checkout(double amount) {
        gateway.processPayment(amount); // OCP: add new gateway, no change here
    }
}
```

---

### Inheritance → Supports LSP (when used correctly)

Inheritance creates IS-A relationships. LSP says inheritance must be done **honestly** — a subclass must truly be a valid substitute for its parent.

```java
// Base class defines a contract
public abstract class Shape {
    public abstract double area();

    // Template method — subclasses extend, not break behavior
    public void printArea() {
        System.out.println("Area: " + area());
    }
}

// Each subclass HONORS the contract — substitution is safe
public class Circle extends Shape {
    private double radius;
    public Circle(double radius) { this.radius = radius; }
    public double area() { return Math.PI * radius * radius; }
}

public class Rectangle extends Shape {
    private double w, h;
    public Rectangle(double w, double h) { this.w = w; this.h = h; }
    public double area() { return w * h; }
}

// Works for ANY Shape — Liskov guaranteed
public class AreaPrinter {
    public void print(Shape shape) {
        shape.printArea(); // safe to substitute Circle, Rectangle, Triangle...
    }
}
```

**Bad inheritance breaks LSP** — always ask: *"Can this subclass fully honor the parent's behavior?"*

---

### Polymorphism → Supports OCP + ISP

Polymorphism allows the same method call to behave differently based on the actual object — enabling you to add new types **without changing existing code**.

```java
// Polymorphism powering OCP
public interface Notification {
    void send(String message);
}

public class EmailNotification implements Notification {
    public void send(String message) { System.out.println("Email: " + message); }
}

public class SMSNotification implements Notification {
    public void send(String message) { System.out.println("SMS: " + message); }
}

public class PushNotification implements Notification {
    public void send(String message) { System.out.println("Push: " + message); }
}

// Polymorphic call — no if/else chains, open for extension
public class NotificationService {
    public void notify(List<Notification> channels, String message) {
        channels.forEach(channel -> channel.send(message)); // polymorphism at work
    }
}
```

---

### OOP → SOLID Mapping Summary

| OOP Pillar | SOLID Principle(s) it enables |
|------------|-------------------------------|
| Encapsulation | S — Single Responsibility |
| Abstraction | O — Open/Closed, D — Dependency Inversion |
| Inheritance | L — Liskov Substitution |
| Polymorphism | O — Open/Closed, I — Interface Segregation |

---

## How Spring Framework Implements SOLID

Spring is essentially **SOLID made into a framework**. Every major Spring feature maps to a SOLID principle.

---

### Spring + SRP — Component Stereotypes (`@Service`, `@Repository`, `@Controller`)

Spring enforces SRP structurally through its **layered architecture** and stereotype annotations. Each layer has one responsibility.

```java
// @Controller — handles HTTP only (SRP)
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<Order> create(@RequestBody OrderRequest req) {
        return ResponseEntity.ok(orderService.placeOrder(req));
    }
}

// @Service — handles business logic only (SRP)
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    public OrderService(OrderRepository repo, PaymentService payment) {
        this.orderRepository = repo;
        this.paymentService = payment;
    }

    public Order placeOrder(OrderRequest req) {
        paymentService.charge(req.getAmount());
        Order order = new Order(req);
        return orderRepository.save(order);
    }
}

// @Repository — handles DB only (SRP)
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerId(Long customerId);
}
```

Each class has **one reason to change** — controller changes only when HTTP interface changes, service when business logic changes, repository when data access changes.

---

### Spring + OCP — `@Conditional`, Profiles, and Strategy Beans

Spring lets you **extend behavior by adding new beans/configurations** without modifying existing code.

```java
// Define abstraction
public interface StorageService {
    void upload(String fileName, byte[] data);
}

// Two implementations — closed for modification, open for extension
@Service
@Profile("local")
public class LocalStorageService implements StorageService {
    public void upload(String fileName, byte[] data) {
        System.out.println("Storing locally: " + fileName);
    }
}

@Service
@Profile("cloud")
public class S3StorageService implements StorageService {
    public void upload(String fileName, byte[] data) {
        System.out.println("Uploading to S3: " + fileName);
    }
}

// High-level service never changes when storage strategy changes
@Service
public class FileUploadService {
    private final StorageService storageService;

    public FileUploadService(StorageService storageService) {
        this.storageService = storageService; // injected by Spring based on profile
    }

    public void handleUpload(String name, byte[] data) {
        storageService.upload(name, data);
    }
}
```

Switch from local to S3 by changing the Spring profile — **zero code changes** in `FileUploadService`.

---

### Spring + LSP — Spring Data JPA & Template Pattern

Spring's `JpaRepository` is a perfect LSP example. Every repository substitutes the parent interface without breaking contracts.

```java
// Spring guarantees: any JpaRepository can be substituted safely
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByCategory(String category);
}

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}

// Both can be used anywhere a CrudRepository is expected — LSP holds
@Service
public class DataService {
    public <T> long count(JpaRepository<T, ?> repo) {
        return repo.count(); // works for Product, User, Order repos — LSP
    }
}
```

Spring AOP proxies also honor LSP — a proxied bean behaves identically to the original for callers.

---

### Spring + ISP — Fine-grained Repository Interfaces

Spring Data offers **specific interfaces** so you don't force classes to implement what they don't need.

```java
// Spring provides segregated interfaces — pick what you need:
// CrudRepository                — basic CRUD only
// PagingAndSortingRepository    — adds paging/sorting
// JpaRepository                 — full JPA features

// Read-only use case — only needs a subset (ISP)
public interface ReportRepository extends Repository<Order, Long> {
    @Query("SELECT o FROM Order o WHERE o.status = 'COMPLETED'")
    List<Order> findCompletedOrders();
    // No save/delete methods — not needed, not forced
}

// Full CRUD use case
public interface OrderRepository extends JpaRepository<Order, Long> {
    // gets everything — save, delete, find, page, sort
}
```

You can also design your own segregated service interfaces:

```java
// Fat interface (bad)
public interface UserService {
    User findById(Long id);
    void save(User user);
    void delete(Long id);
    void sendWelcomeEmail(User user);  // shouldn't be here
    void generateReport();             // shouldn't be here
}

// Segregated (good — Spring beans for each)
public interface UserQueryService {
    User findById(Long id);
}

public interface UserCommandService {
    void save(User user);
    void delete(Long id);
}

@Service
public class UserQueryServiceImpl implements UserQueryService {
    public User findById(Long id) { /* ... */ return null; }
}

@Service
public class UserCommandServiceImpl implements UserCommandService {
    public void save(User user) { /* ... */ }
    public void delete(Long id) { /* ... */ }
}
```

---

### Spring + DIP — Dependency Injection (The Heart of Spring)

This is where **Spring contributes most powerfully to SOLID**. Spring's entire IoC container is built around DIP.

```java
// ===== Abstraction =====
public interface EmailService {
    void send(String to, String subject, String body);
}

public interface SMSService {
    void sendSMS(String phone, String message);
}

// ===== Low-level implementations =====
@Service("gmailService")
public class GmailEmailService implements EmailService {
    public void send(String to, String subject, String body) {
        System.out.println("Sending via Gmail to: " + to);
    }
}

@Service("sendgridService")
public class SendGridEmailService implements EmailService {
    public void send(String to, String subject, String body) {
        System.out.println("Sending via SendGrid to: " + to);
    }
}

// ===== High-level module — depends on abstraction, Spring injects the impl =====
@Service
public class UserRegistrationService {
    private final EmailService emailService; // interface, not concrete class
    private final SMSService smsService;

    // Constructor injection — Spring wires everything automatically
    @Autowired
    public UserRegistrationService(
            @Qualifier("sendgridService") EmailService emailService,
            SMSService smsService) {
        this.emailService = emailService;
        this.smsService = smsService;
    }

    public void register(String email, String phone) {
        // business logic
        emailService.send(email, "Welcome!", "Thanks for joining.");
        smsService.sendSMS(phone, "Your account is active.");
        // Switch email provider? Change @Qualifier or config — zero logic change
    }
}
```

Spring also supports **externalizing the decision** to `application.properties`:

```java
// application.properties
// app.email.provider=sendgrid

// Config class
@Configuration
public class EmailConfig {
    @Value("${app.email.provider}")
    private String provider;

    @Bean
    public EmailService emailService() {
        return switch (provider) {
            case "sendgrid" -> new SendGridEmailService();
            case "gmail"    -> new GmailEmailService();
            default -> throw new IllegalArgumentException("Unknown provider: " + provider);
        };
    }
}
```

Switch email providers **without touching a single line of business code**.

---

## Complete Mapping: OOP + Spring → SOLID

| SOLID Principle | OOP Feature | Spring Feature |
|-----------------|-------------|----------------|
| **S**RP | Encapsulation | `@Controller`, `@Service`, `@Repository` layers |
| **O**CP | Abstraction + Polymorphism | `@Profile`, `@Conditional`, Strategy beans |
| **L**SP | Inheritance (honest) | `JpaRepository` hierarchy, AOP proxies |
| **I**SP | Interface segregation | `CrudRepository` vs `JpaRepository`, custom interfaces |
| **D**IP | Abstraction | `@Autowired`, `@Qualifier`, IoC Container |

---

## The Big Picture

```
SOLID Principles
      |
      |-- OOP provides the LANGUAGE (interfaces, classes, inheritance)
      |
      |-- Spring provides the FRAMEWORK (wiring, lifecycle, configuration)
      |
      Both together = Maintainable, Testable, Extensible Enterprise Java
```

- **OOP gives you the tools** (abstract classes, interfaces, polymorphism) to express SOLID.
- **Spring automates the wiring** so you never manually `new` dependencies — the IoC container is essentially DIP enforced at framework level, which cascades and enforces the rest of SOLID naturally.

---

*Reference: Robert C. Martin — "Design Principles and Design Patterns" (2000) | "Agile Software Development, Principles, Patterns, and Practices" (2002)*
