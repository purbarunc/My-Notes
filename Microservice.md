# Microservice

A **microservice** is a small, independently deployable service that handles a single business capability — for example, user authentication, payments, or search — and communicates with other services over a network.

## Key Characteristics

- **Independent deployment** — each service can be built, tested, and shipped on its own release cycle
- **Single responsibility** — scoped to one business capability or domain
- **Owns its data** — each service manages its own database rather than sharing one
- **Decentralized** — teams can choose different languages, frameworks, and tools per service
- **Loose coupling** — services interact through well-defined APIs (REST, gRPC, or messaging queues)

## Microservices vs. Monolith

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| Deployment | Single unit, all at once | Per service, independently |
| Scaling | Scale the whole app | Scale only hot services |
| Tech stack | One stack for everything | Polyglot per service |
| Data | Shared database | Database per service |
| Failure | One bug can down the app | Failures can be isolated |
| Complexity | Simple to run locally | Requires orchestration, discovery, observability |

## Communication Patterns

- **Synchronous** — REST/HTTP, gRPC (request/response)
- **Asynchronous** — message brokers like Kafka, RabbitMQ (event-driven)

## Common Infrastructure Pieces

- **API gateway** — single entry point, routing, auth, rate limiting
- **Service discovery** — how services find each other (e.g., Consul, Kubernetes DNS)
- **Container orchestration** — Kubernetes, Docker Swarm
- **Observability** — distributed tracing, centralized logging, metrics

## Trade-offs

**Benefits:** team autonomy, independent scaling, fault isolation, technology flexibility.

**Costs:** distributed systems complexity — network latency, eventual consistency, harder debugging, more infrastructure to operate, and the need for mature DevOps practices.

---

## Kubernetes vs Spring Cloud: API Gateway & Load Balancing

### Spring Cloud approach (application-level)

Spring Cloud implements these concerns **inside the application**, as libraries embedded in each service:

- **API Gateway** — Spring Cloud Gateway (or Zuul) runs as its own Spring Boot app; routing rules, filters, and auth logic are written in Java/config
- **Load Balancing** — Spring Cloud LoadBalancer (or Ribbon) runs *client-side*, inside each service; every service resolves and picks an instance itself
- **Service Discovery** — Eureka/Consul client embedded in each service; services register themselves and query the registry

This means every service carries the weight of these libraries, and your routing/LB logic lives in application code.

### Kubernetes approach (infrastructure-level)

Kubernetes moves these concerns **out of the application** into the platform:

| Concern | Kubernetes Mechanism |
|---------|----------------------|
| Load Balancing | `Service` (kube-proxy, iptables/IPVS) handles it transparently — app knows nothing |
| API Gateway / Ingress | `Ingress` resource + an Ingress Controller (NGINX, Traefik, AWS ALB) |
| Advanced Gateway | `Gateway API` (newer) or tools like Kong, Istio built on top of K8s |
| Service Discovery | Built-in DNS — `http://payment-service` just works, no registry client needed |

### Key Differences

| Aspect | Spring Cloud | Kubernetes |
|--------|--------------|------------|
| Where it lives | Application (libraries) | Infrastructure (platform) |
| Language | Java only | Language-agnostic |
| Load balancing | Client-side (each service decides) | Server-side (platform decides) |
| Gateway logic | Code/config in a Spring app | YAML/CRD config + Ingress Controller |
| Service discovery | Eureka/Consul client in every app | kube-dns, no client needed |
| Ops burden | Each team owns it | Platform team owns it once |
| Fine-grained control | High (full Java code) | Moderate (config-driven, extensible via plugins) |

### When to Prefer Which

- **Spring Cloud** — you're on bare metal/VMs without K8s, need highly custom routing logic in Java, or already have a deep Spring ecosystem
- **Kubernetes** — you're containerized; you want to remove cross-cutting concerns from app code and let teams focus on business logic; polyglot services

In practice many teams use **both** — Kubernetes handles infrastructure-level LB and ingress, while Spring Cloud Gateway adds application-level concerns (auth, rate limiting, response transformation) for services that need it.

---

## Which is Better for Enterprise Applications?

**Short answer: Kubernetes-first, with Spring Cloud Gateway for application-level concerns where needed.**

### Why Kubernetes Wins at the Enterprise Level

1. **Polyglot teams** — Enterprises rarely run only Java. Kubernetes works for Node, Python, .NET, Go equally. Spring Cloud is Java-only.
2. **Ops standardization** — One platform team manages ingress/LB/discovery for all services. With Spring Cloud, every team configures and maintains their own registry clients, adding inconsistency at scale.
3. **Zero code for infrastructure** — Developers don't write routing or LB code; it's declared in YAML and owned by the platform. Reduces bugs and onboarding friction.
4. **Cloud-native & vendor support** — AWS (ALB), GCP (Cloud Load Balancing), Azure (Application Gateway) all integrate natively with Kubernetes Ingress. Enterprise support contracts exist for this stack.
5. **Service mesh ready** — Istio, Linkerd layer on top of K8s to add mTLS, observability, circuit breaking — with zero app code changes.
6. **Scalability** — Kubernetes HPA (Horizontal Pod Autoscaler) scales services automatically based on CPU/memory/custom metrics.

### Where Spring Cloud Still Has a Place in Enterprise

| Use Case | Spring Cloud Value |
|---|---|
| Complex routing logic | Custom filters in Java are more powerful than Ingress annotations |
| Legacy Spring Boot apps | Already have Eureka/Ribbon; migration cost is high |
| On-premise without K8s | Spring Cloud works on VMs without container orchestration |
| Circuit breaking / Resilience | Resilience4j (Spring Cloud) gives fine-grained control |
| Developer local testing | Easier to run Spring Cloud stack locally vs a full K8s cluster |

### Recommended Enterprise Architecture

```
Internet
    |
[Kubernetes Ingress / API Gateway]  ← NGINX, Kong, AWS ALB
    |
[Spring Cloud Gateway]              ← Auth, rate limiting, request transformation (optional)
    |
[Microservices in K8s Pods]         ← communicate via kube-dns
    |
[Service Mesh - Istio/Linkerd]      ← mTLS, tracing, circuit breaking (optional)
```

- **K8s Ingress** handles external traffic routing, TLS termination, LB
- **Spring Cloud Gateway** (if needed) adds business-level API policies
- **kube-dns** replaces Eureka entirely
- **Istio/Linkerd** replaces Ribbon/Hystrix for resilience and observability

### Bottom Line

| Factor | Verdict |
|---|---|
| New greenfield enterprise project | Kubernetes-first |
| Existing Spring Boot monolith being broken apart | Spring Cloud as stepping stone, migrate to K8s over time |
| Regulated industries (banking, insurance) | Kubernetes + Istio (audit trails, mTLS, zero-trust) |
| Small team, no K8s expertise | Spring Cloud initially, plan K8s adoption |

---

## Distributed Transactions in Microservices

### Why `@Transactional` Does NOT Work

`@Transactional` in Spring relies on **ACID transactions** managed by a single database connection (via a local transaction manager like JPA/JDBC). It works within one service talking to one database.

In microservices:

```
Order Service (DB1)  →  Payment Service (DB2)  →  Inventory Service (DB3)
```

- Each service has its **own database** — there is no single connection to wrap in a transaction
- `@Transactional` only rolls back **within its own service's DB**
- If Payment succeeds but Inventory fails, `@Transactional` on Order Service **cannot roll back Payment** — it has no knowledge of it
- You'd need a **distributed transaction manager** (XA/2PC), which is slow, complex, and breaks microservice independence

---

### Solutions for Distributed Transactions

#### 1. SAGA Pattern (Most Widely Used)

Break the transaction into a **sequence of local transactions**, each publishing an event. If one step fails, compensating transactions undo previous steps.

**a) Choreography SAGA** (event-driven, decentralized)
```
Order Service       → emits OrderCreated
  → Payment Service  → emits PaymentProcessed (or PaymentFailed)
    → Inventory Service → emits InventoryReserved (or InventoryFailed)
```
- No central coordinator
- Services react to events via Kafka/RabbitMQ
- Simple, but hard to track overall flow

**b) Orchestration SAGA** (centralized coordinator)
```
SAGA Orchestrator
  → calls Payment Service
  → calls Inventory Service
  → if failure → calls CompensatePay, CompensateInventory
```
- A dedicated orchestrator (e.g., Temporal, Axon, Conductor) manages the flow
- Easier to visualize and debug
- Single point of logic

---

#### 2. Outbox Pattern (Reliable Event Publishing)

Solves the problem: *"What if a service saves to DB but crashes before publishing the event?"*

```
Order Service:
  1. Write order to Orders table    } in ONE local @Transactional
  2. Write event to Outbox table    }
  3. A poller/CDC (Debezium) reads Outbox and publishes to Kafka
```

- Guarantees **at-least-once** event delivery
- Commonly paired with SAGA

---

#### 3. Two-Phase Commit / XA Transactions (Avoid in Microservices)

- A coordinator asks all services to "prepare", then "commit"
- **Problems:** tightly couples services, blocks resources during prepare phase, kills performance, single point of failure
- Only viable in monoliths or tightly coupled legacy systems — **not recommended for microservices**

---

#### 4. Eventual Consistency (Accept It)

- Instead of atomicity, design for **eventual consistency**
- Use idempotent operations, retry mechanisms, and compensating actions
- Most modern enterprise systems (Amazon, Netflix) embrace this model

---

### Comparison

| Pattern | Consistency | Complexity | Best For |
|---|---|---|---|
| `@Transactional` | Strong (ACID) | Low | Single service, single DB |
| XA / 2PC | Strong | Very High | Legacy, avoid in microservices |
| Choreography SAGA | Eventual | Medium | Simple flows, event-driven teams |
| Orchestration SAGA | Eventual | Medium-High | Complex flows, clear visibility needed |
| Outbox Pattern | Eventual (reliable) | Medium | Reliable event publishing |

---

### Real-World Stack

```
Order Service
  └── Local @Transactional (order DB + outbox table)
        └── Debezium (CDC) → Kafka
              └── SAGA Orchestrator (Temporal / Axon)
                    ├── Payment Service (compensate on failure)
                    └── Inventory Service (compensate on failure)
```

---

## What Happens If One Microservice Is Down?

### The Problem — Cascading Failures

When a microservice goes down, the danger isn't just that one feature stops working — it's the **domino effect**:

```
User Request
    → Order Service (OK)
        → Payment Service (DOWN) ← times out after 30s
            → Order Service thread pool exhausted
                → Order Service becomes slow/unresponsive
                    → Gateway times out
                        → ALL users affected
```

This is called a **cascading failure** — one unhealthy service drags down the entire system.

---

### How Kubernetes Helps

Kubernetes has **multiple layers of defense** against a service going down:

#### 1. Health Checks (Liveness & Readiness Probes)

Kubernetes continuously checks if a pod is alive and ready:

```yaml
livenessProbe:   # Is the app alive? If not → restart the pod
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:  # Is the app ready to serve traffic? If not → remove from load balancer
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 5
```

- **Liveness probe fails** → K8s kills and restarts the pod automatically
- **Readiness probe fails** → K8s stops sending traffic to that pod (but doesn't kill it)

#### 2. Automatic Restart (Self-Healing)

If a pod crashes, Kubernetes restarts it automatically based on the `restartPolicy`. It uses exponential backoff (10s → 20s → 40s...) to avoid restart loops.

#### 3. ReplicaSet — Multiple Instances

```yaml
replicas: 3  # Always keep 3 instances running
```

- If one pod dies, the other 2 keep serving traffic
- Kubernetes immediately spins up a replacement pod to restore 3 replicas
- No single point of failure

#### 4. Horizontal Pod Autoscaler (HPA)

If a service is under heavy load (causing slowdowns that look like failures):

```yaml
minReplicas: 2
maxReplicas: 10
targetCPUUtilizationPercentage: 70
```

K8s automatically scales up pods when load increases, preventing overload-induced failures.

#### 5. Rolling Updates (Zero Downtime Deployments)

When deploying a new version, K8s replaces pods gradually — never taking all instances down at once:

```
Old Pod 1 (running) → New Pod 1 starts → Old Pod 1 terminates
Old Pod 2 (running) → New Pod 2 starts → Old Pod 2 terminates
```

No downtime during deployments.

---

### What Kubernetes CANNOT Do — App-Level Resilience

Kubernetes handles **infrastructure-level** failures. For **application-level** resilience (e.g., Payment Service is up but responding slowly), you need patterns in your code:

| Problem | Solution | Tool |
|---|---|---|
| Downstream service slow/down | **Circuit Breaker** — stop calling a failing service | Resilience4j, Istio |
| Retry on transient failure | **Retry with backoff** | Resilience4j, Spring Retry |
| Prevent thread pool exhaustion | **Bulkhead** — isolate thread pools per service | Resilience4j |
| Slow response acceptable | **Timeout** — fail fast instead of waiting | Resilience4j, Feign |
| Return something useful when down | **Fallback** — return cached/default response | Resilience4j |

#### Circuit Breaker States

```
CLOSED (normal) → too many failures → OPEN (stop calling)
    ↑                                        ↓
    └──── half-open (test one call) ←────────┘
```

---

### Complete Resilience Stack

```
Kubernetes Layer (Infrastructure)
  ├── Liveness Probe    → auto-restart crashed pods
  ├── Readiness Probe   → remove bad pods from LB
  ├── ReplicaSet        → always N healthy instances
  ├── HPA               → scale out under load
  └── Rolling Updates   → zero downtime deploys

Application Layer (Code / Service Mesh)
  ├── Circuit Breaker   → stop cascading failures
  ├── Retry             → handle transient errors
  ├── Timeout           → fail fast
  ├── Bulkhead          → isolate failure blast radius
  └── Fallback          → degrade gracefully
```

**Both layers are needed** — Kubernetes keeps services running, but your application must be designed to tolerate downstream failures gracefully.

---

## Circuit Breaker and Bulkhead Pattern (In Simple Terms)

### Circuit Breaker Pattern

#### Think of it like an electrical circuit breaker at home

In your home, if there's a power surge or short circuit, the breaker **trips and cuts off power** to protect your appliances. You don't keep letting electricity flow into a broken wire.

Same idea in microservices:

```
Normal (CLOSED):
  Order Service → calls → Payment Service ✅ (working fine)

Too many failures (OPEN):
  Order Service → X → Payment Service ❌ (stop calling, it's down!)
  Order Service → returns fallback response immediately

Testing recovery (HALF-OPEN):
  Order Service → sends 1 test call → Payment Service
    ✅ success → go back to CLOSED (resume normal calls)
    ❌ failure → stay OPEN (still broken)
```

**Without Circuit Breaker:**
- Order Service keeps calling Payment Service even though it's down
- Each call waits 30 seconds to time out
- Threads pile up → Order Service runs out of threads → Order Service also goes down

**With Circuit Breaker:**
- After 5 failures, stop calling Payment Service immediately
- Return a fallback response (e.g., "Payment service temporarily unavailable, try again later")
- Automatically test recovery after 60 seconds
- **Fails fast** instead of waiting — protects the caller

---

### Bulkhead Pattern

#### Think of it like compartments in a ship

A ship's hull is divided into **watertight compartments (bulkheads)**. If one compartment floods, the others stay dry — the ship doesn't sink entirely.

```
Without Bulkhead:
  One shared thread pool (10 threads)

  Payment calls use all 10 threads (Payment Service is slow)
  → No threads left for Inventory calls
  → No threads left for User calls
  → Entire Order Service is frozen ❌

With Bulkhead:
  Payment Service   → own pool (3 threads)
  Inventory Service → own pool (3 threads)
  User Service      → own pool (3 threads)

  Payment Service goes slow → only its 3 threads are stuck
  → Inventory and User calls still work fine ✅
```

**In simple words:**
- Without Bulkhead: one slow neighbour floods the whole house
- With Bulkhead: each neighbour has their own water pipe — one leak doesn't affect others

---

### Together in Real Life

Imagine an e-commerce checkout:

```
Checkout Service calls:
  ├── Payment Service      (Bulkhead: 5 threads, Circuit Breaker: opens after 3 failures)
  ├── Inventory Service    (Bulkhead: 5 threads, Circuit Breaker: opens after 3 failures)
  └── Notification Service (Bulkhead: 3 threads, Circuit Breaker: opens after 5 failures)
```

Payment goes down →
- Circuit Breaker opens → stops calling Payment immediately
- Bulkhead ensures Inventory & Notification threads are unaffected
- User gets: *"Order placed, payment will be retried"* (fallback)
- Everything else keeps working normally ✅

---

| Pattern | Real World Analogy | Protects Against |
|---|---|---|
| Circuit Breaker | Home electrical breaker | Calling a dead service repeatedly |
| Bulkhead | Ship compartments | One slow service starving all others |

---

## API Gateway vs Load Balancer vs Service Discovery

### What Each One Does (Simple Definitions)

| | API Gateway | Load Balancer | Service Discovery |
|---|---|---|---|
| **Job** | Smart traffic cop — routes, secures, transforms requests | Spreads traffic evenly across instances | Phone book — finds where a service lives |
| **Asks** | "What should I do with this request?" | "Which instance should get this request?" | "Where is the Payment Service right now?" |
| **Who uses it** | External clients (browser, mobile app) | Internal and external traffic | Services talking to each other |

---

### API Gateway

Think of it as the **front door of your entire system**:

```
Mobile App / Browser / Third Party
          ↓
    [ API Gateway ]
    ├── Authentication & Authorization (JWT, OAuth)
    ├── Rate Limiting (100 req/min per user)
    ├── Request Routing (/orders → Order Service, /payments → Payment Service)
    ├── SSL Termination (HTTPS → HTTP inside)
    ├── Request/Response Transformation
    ├── Logging & Analytics
    └── Caching
          ↓
   Microservices
```

- Clients talk to **one URL** — gateway handles the rest
- Shields internal services from the outside world
- Examples: Kong, AWS API Gateway, NGINX, Spring Cloud Gateway

---

### Load Balancer

Think of it as a **traffic distributor** — it doesn't care what the request is, just spreads load:

```
          Request
             ↓
    [ Load Balancer ]
    ├── Payment Service Instance 1  (30% load)
    ├── Payment Service Instance 2  (35% load)
    └── Payment Service Instance 3  (35% load)
```

- Algorithms: Round Robin, Least Connections, IP Hash
- **Does NOT** do auth, routing rules, or transformation — just distributes
- Two types:
  - **L4 (Transport)** — routes by IP/TCP — very fast, no content inspection
  - **L7 (Application)** — routes by HTTP headers/URL — smarter, used more commonly
- Examples: AWS ALB/NLB, NGINX, HAProxy, Kubernetes Service

---

### Service Discovery

Think of it as a **live phone book** for your services:

```
Problem without it:
  Order Service wants to call Payment Service
  But Payment Service runs on dynamic IPs in K8s (pods come and go)
  Order Service doesn't know where to find it!

Solution with Service Discovery:
  Payment Service starts → registers itself → "I'm at 10.0.0.5:8080"
  Order Service asks registry → "Where is Payment Service?" → "10.0.0.5:8080"
  Pod dies, new pod starts → re-registers at new IP automatically
```

- Two types:
  - **Client-side** (Spring Cloud / Eureka) — each service queries the registry and picks an instance itself
  - **Server-side** (Kubernetes DNS) — a proxy/platform resolves the address, app just uses a name like `http://payment-service`
- Examples: Eureka, Consul, Kubernetes DNS, AWS Cloud Map

---

### How All 3 Work Together

```
                        INTERNET
                            |
                    [ API Gateway ]           ← Auth, Rate Limiting, Routing
                    Kong / AWS API GW
                            |
                  [ Load Balancer (L7) ]      ← Spread traffic across gateway instances
                    AWS ALB / NGINX
                            |
              ┌─────────────┼──────────────┐
              ↓             ↓              ↓
        Order Service  Payment Service  User Service
              |             |              |
        [ Load Balancer ]  [ Load Balancer ]  [ Load Balancer ]
         (K8s Service)      (K8s Service)      (K8s Service)
              |                  |
      Pod1 Pod2 Pod3       Pod1 Pod2 Pod3
```

When Order Service calls Payment Service:
1. Order Service asks DNS → **Service Discovery** resolves `payment-service` to a K8s Service IP
2. K8s Service → **Load Balancer** distributes to one of the Payment pods
3. External calls first hit **API Gateway** → authenticated → routed → load balanced

---

### How Large Enterprises Handle All 3

#### Typical Enterprise Stack

| Layer | Tool Used | Why |
|---|---|---|
| External API Gateway | Kong, AWS API Gateway, Apigee | Centralized auth, rate limiting, analytics, developer portal |
| Internal API Gateway | Spring Cloud Gateway or Istio | Service-to-service policies, internal routing |
| External Load Balancer | AWS ALB, Azure App Gateway, GCP LB | Cloud-native, integrates with K8s Ingress |
| Internal Load Balancer | Kubernetes Service (ClusterIP) | Automatic, no config needed per service |
| Service Discovery | Kubernetes DNS (kube-dns/CoreDNS) | Built-in, zero client code needed |
| Service Mesh (advanced) | Istio / Linkerd | Combines LB + discovery + circuit breaking + mTLS in one layer |

#### Enterprise Architecture (Full Picture)

```
External Users
      |
[ WAF / DDoS Protection ]          ← AWS Shield, Cloudflare
      |
[ External Load Balancer ]         ← AWS ALB (routes to K8s Ingress)
      |
[ Kubernetes Ingress ]             ← NGINX / Traefik (TLS termination)
      |
[ API Gateway ]                    ← Kong / Apigee (auth, rate limit, routing)
      |
[ Internal Service Mesh ]          ← Istio (mTLS, observability, LB between pods)
      |
[ Microservices in K8s ]
  Order → payment-service (kube-dns resolves) → K8s Service → pods
```

---

### Quick Rule of Thumb

| Concern | Who Handles It |
|---|---|
| External client routing & security | API Gateway |
| Spreading traffic across instances | Load Balancer |
| Finding where a service is | Service Discovery |
| All three internally between services | Kubernetes (Service + CoreDNS) |
| All three with advanced policies | Istio / Service Mesh |

---

## Why Pods Come and Go

Pods are **temporary by design** in Kubernetes. Unlike traditional servers that run 24/7, pods are meant to be disposable. Here are all the reasons they come and go:

### 1. Deployment / Updates
When you deploy a new version of your app:
```
Old Pod (v1) → killed
New Pod (v2) → started at a NEW IP address
```
Kubernetes does this automatically during rolling updates — old pods die, new ones are born.

### 2. Auto-Scaling (HPA)
```
Traffic low  → HPA scales down → pods killed to save resources
Traffic high → HPA scales up   → new pods created on any available node
```
Pods are created and destroyed based on load — constantly.

### 3. Pod Crashes / Restarts
If your app throws an unhandled exception, runs out of memory, or fails a liveness probe:
```
Pod crashes → Kubernetes kills it → restarts a new pod → new IP assigned
```
The new pod gets a **brand new IP** — it's not the same pod resurrected.

### 4. Node Failure
If the underlying server (node) goes down:
```
Node dies → all pods on that node die
          → Kubernetes reschedules them on healthy nodes
          → new pods, new IPs
```

### 5. Resource Eviction
Kubernetes can evict (forcefully remove) pods from a node if the node is running low on memory or CPU:
```
Node running low on memory
→ Kubernetes evicts lower-priority pods
→ They restart on another node with a new IP
```

### 6. Manual Scaling / Rollback
Developers or CI/CD pipelines scale services up/down or roll back bad deployments — pods are replaced constantly.

---

### Why This Breaks Static IPs

In traditional servers you'd hardcode:
```
payment-service = 192.168.1.10:8080  ← fixed forever
```

In Kubernetes, after any of the above events:
```
payment-service pod 1 = 10.0.0.5:8080   (today)
payment-service pod 1 = 10.0.0.23:8080  (after restart — different IP!)
```

Hardcoded IPs become stale immediately. This is **exactly why Service Discovery exists** — instead of tracking IPs, services just ask:

```
"Where is payment-service RIGHT NOW?"
→ Kubernetes DNS always returns the current live address
```

---

### The Simple Analogy

Think of pods like **hotel guests**, not residents:
- A resident has a permanent address you can write down
- A hotel guest checks in and out — their room number changes every visit
- The hotel front desk (Service Discovery) always knows which room they're currently in

Kubernetes embraces the **"cattle not pets"** philosophy — pods are treated as disposable units, not precious permanent machines. This gives you resilience and scalability, but requires dynamic service discovery to keep everything connected.

---

## Why an Unhandled Exception Can Crash a Pod

### First, Understand What a Pod Really Is

A pod is just a **wrapper around a running process** (your app). When Kubernetes starts a pod, it essentially runs:

```
java -jar payment-service.jar   ← this IS your pod
```

The pod is alive **only as long as this process is alive**. The moment the process exits — for any reason — the pod is considered crashed.

---

### What Happens With an Unhandled Exception

**In a web app (Spring Boot etc.):** the framework catches it, returns a 500 error, thread goes back to pool — app keeps running. ✅

**BUT — in certain situations, the JVM process itself exits:**

#### 1. Exception in Main Thread
```java
public static void main(String[] args) {
    throw new RuntimeException("startup failed!");
    // JVM exits immediately — process dead — pod crashes
}
```
App never even starts up fully → pod enters `CrashLoopBackOff`

#### 2. OutOfMemoryError (Most Common in Production)
```
App leaks memory slowly...
Heap fills up...
JVM throws OutOfMemoryError
→ JVM process exits
→ Pod crashes
```
This is the #1 real-world cause of pod crashes.

#### 3. StackOverflowError
```java
void infinite() { infinite(); }  // infinite recursion
// StackOverflowError → JVM exits → pod crashes
```

#### 4. `System.exit()` Called Accidentally
```java
catch (Exception e) {
    System.exit(1);  // kills the entire JVM process
}
```

#### 5. Exception in a Background Thread
```java
new Thread(() -> {
    throw new RuntimeException("background thread exploded!");
    // If it's a critical thread (e.g., DB connection pool monitor),
    // the app may become unusable even though the process is still running
}).start();
```

---

### The Full Crash → Restart Cycle in Kubernetes

```
1. JVM process exits (any reason above)
         ↓
2. Container runtime (Docker) sees process exited with non-zero code
         ↓
3. Kubernetes marks pod as "Failed"
         ↓
4. Kubernetes restarts pod (based on restartPolicy: Always)
         ↓
5. New pod starts — new IP address assigned
         ↓
6. If it keeps crashing → CrashLoopBackOff
   (K8s waits 10s → 20s → 40s → 80s... before each restart)
```

---

### CrashLoopBackOff — What It Means

```
kubectl get pods

NAME                    READY   STATUS             RESTARTS
payment-service-xyz     0/1     CrashLoopBackOff   5
```

This means:
- Pod starts → crashes → K8s restarts it → crashes again → repeat
- K8s keeps trying but backs off to avoid thrashing
- Usually caused by: bad config, missing env variable, DB not reachable on startup, or an unhandled exception in `main()`

---

### Liveness Probe — The Safety Net

Even if the JVM process doesn't exit but the app is **stuck/frozen** (e.g., deadlock), Kubernetes uses the liveness probe to detect it:

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  failureThreshold: 3
  periodSeconds: 10
```

```
App deadlocked → can't respond to /health → probe fails 3 times
→ Kubernetes kills and restarts the pod
→ Even though JVM process technically never "crashed"
```

---

### Simple Summary

| Cause | Does Process Exit? | Pod Crashes? |
|---|---|---|
| Handled exception (Spring catches it) | No | No |
| Unhandled exception in main thread | Yes | Yes |
| OutOfMemoryError | Yes | Yes |
| StackOverflowError | Yes | Yes |
| `System.exit(1)` | Yes | Yes |
| Deadlock / frozen app | No | Yes (liveness probe kills it) |
| Background thread exception | No (usually) | No (but app may be broken) |

**The key rule:** if the JVM process exits → container exits → pod crashes → Kubernetes restarts it.

---

## Service Boundaries (In Simple Terms)

### The Simple Idea

A service boundary is the **line you draw around what one microservice owns and is responsible for**. Everything inside that line belongs to that service — its data, its logic, its decisions. Nothing outside crosses that line without asking.

Think of it like **departments in a company:**

```
HR Department          Finance Department       IT Department
─────────────          ──────────────────       ─────────────
Owns: employee data    Owns: salary data        Owns: systems data
Does: hiring, firing   Does: payroll, budgets   Does: infra, support
Has: its own files     Has: its own ledgers     Has: its own tools

HR doesn't go into Finance's files directly.
HR asks Finance: "Please process this salary."
Finance does it their way and responds.
```

That separation between departments = **service boundaries** in microservices.

---

### In a Real App — E-Commerce Example

Bad design — no clear boundaries:
```
One giant service knows everything:
  - User details, Order history, Payment info
  - Inventory stock, Delivery address, Notifications
→ Change one thing, break everything
→ One team steps on another team's code
```

Good design — clear boundaries:
```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  User Service   │  │  Order Service  │  │ Payment Service │
│─────────────────│  │─────────────────│  │─────────────────│
│ Owns:           │  │ Owns:           │  │ Owns:           │
│  - user profile │  │  - order data   │  │  - transactions │
│  - login/auth   │  │  - order status │  │  - refunds      │
│  - address      │  │  - cart         │  │  - invoices     │
│                 │  │                 │  │                 │
│ Its own DB      │  │ Its own DB      │  │ Its own DB      │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

Each service **owns its data completely** — no other service touches its database directly.

---

### The Golden Rules of Service Boundaries

#### Rule 1: Own Your Data
```
❌ Wrong — Order Service directly queries Payment DB:
   SELECT * FROM payment_db.transactions WHERE order_id = 123

✅ Right — Order Service asks Payment Service via API:
   GET /payments?orderId=123
   Payment Service queries its own DB and responds
```

#### Rule 2: Do One Thing Well
```
❌ Wrong — User Service also handles orders and refunds:
   UserService.createOrder()
   UserService.getUserProfile()
   UserService.processRefund()

✅ Right — each service has one job:
   UserService.getUserProfile()
   OrderService.createOrder()
   PaymentService.processRefund()
```

#### Rule 3: Communicate Only Through APIs
```
Services are like black boxes:
  ┌───────────────────┐
  │   Order Service   │
  │                   │
  │  [internal logic] │ ← nobody sees inside
  │  [internal DB]    │ ← nobody touches directly
  │                   │
  └────────┬──────────┘
           │ API (REST/gRPC/Events)
           ↓
  ┌───────────────────┐
  │ Payment Service   │
  └───────────────────┘
```

---

### How Do You Decide Where to Draw the Line?

The best guide is **Domain-Driven Design (DDD)** — draw boundaries around real-world business concepts:

| Ask This | Example Answer | Service |
|---|---|---|
| What changes together? | Order + cart items always change together | One service |
| What is owned by one team? | Finance team owns payments | Payment Service |
| What has its own business rules? | Inventory has stock rules | Inventory Service |
| What could work independently? | Notifications don't need Order logic | Notification Service |

---

### Signs You Got the Boundaries Wrong

| Symptom | Problem |
|---|---|
| Changing Service A always requires changing Service B | Boundary too tight — they're coupled |
| Service A directly queries Service B's database | No boundary at all |
| One service does 20 different things | Boundary too wide — needs splitting |
| Services can never be deployed independently | Wrong boundaries |
| Teams constantly step on each other's code | Boundary doesn't match team ownership |

---

### The Simplest Analogy

Think of each microservice as a **restaurant kitchen station:**

```
Grill Station    Salad Station    Dessert Station    Drinks Station
─────────────    ─────────────    ───────────────    ──────────────
Owns: grill      Owns: greens     Owns: sweets       Owns: beverages
Does: meat/fish  Does: salads     Does: cakes/ice    Does: cocktails

The grill chef doesn't reach into the salad station.
They communicate: "I need a side salad for table 5"
Each station works independently, delivers their part,
the waiter (API Gateway) assembles and serves the full meal.
```

**Service boundary** = the wall between each kitchen station.

---

## Observability in Microservices

### The Problem

In a monolith, when something goes wrong you look at one log file. In microservices:

```
User reports: "My order failed"
Which service failed? Order? Payment? Inventory? Notification?
A single request touches 6 services — where did it break?
```

Observability is the ability to understand **what is happening inside your system** from the outside. It has three pillars:

### The Three Pillars

```
┌─────────────────────────────────────────────────────┐
│                   OBSERVABILITY                      │
│                                                      │
│  LOGS          METRICS           TRACES              │
│  ─────         ───────           ──────              │
│  What          How much /        Where did           │
│  happened?     how fast?         time go?            │
│                                                      │
│  ELK Stack     Prometheus        Jaeger / Zipkin     │
│  Loki          Grafana           OpenTelemetry       │
└─────────────────────────────────────────────────────┘
```

---

### 1. Distributed Tracing (Traces)

Assigns a **unique Trace ID** to every request — it travels across all services so you can follow the full journey:

```
User → Order Service → Payment Service → Inventory Service
         TraceID: abc123

Timeline:
  Order Service:    0ms  → 12ms   (12ms)
  Payment Service: 12ms  → 187ms  (175ms) ← SLOW, investigate here
  Inventory Service: 187ms → 195ms (8ms)
  Total: 195ms
```

- Tools: **Jaeger, Zipkin, AWS X-Ray**
- Standard: **OpenTelemetry** (vendor-neutral, works with all tools)
- Spring Boot: add `spring-boot-starter-actuator` + OpenTelemetry agent

---

### 2. Centralized Logging

Each service writes logs — centralized logging aggregates all of them into one searchable place:

```
Order Service logs    ──┐
Payment Service logs  ──┼──→ [ Logstash / Fluentd ] → [ Elasticsearch ] → [ Kibana ]
Inventory logs        ──┘                                                  (search UI)
```

**ELK Stack:** Elasticsearch + Logstash + Kibana
**EFK Stack:** Elasticsearch + Fluentd + Kibana (more K8s-friendly)
**Loki + Grafana:** lightweight alternative, popular in K8s

Key practice — always include **TraceID** in every log line:
```
[TraceID: abc123] Payment processed for order 456
[TraceID: abc123] Inventory reserved for order 456
```
Now you can search by TraceID and see all logs across all services for one request.

---

### 3. Metrics

Numbers that tell you the **health and performance** of your system:

```
RED Method (for every service):
  Rate     — requests per second
  Errors   — error rate (%)
  Duration — response time (p50, p95, p99)

USE Method (for infrastructure):
  Utilization — CPU, memory usage
  Saturation  — queue depth, thread pool fullness
  Errors      — hardware/network errors
```

- **Prometheus** scrapes metrics from services
- **Grafana** visualizes them as dashboards
- Spring Boot Actuator exposes `/actuator/prometheus` endpoint automatically

---

### Observability Stack in Enterprise

```
Services (Spring Boot + OpenTelemetry agent)
    │
    ├── Traces  → Jaeger / AWS X-Ray
    ├── Logs    → Fluentd → Elasticsearch → Kibana
    └── Metrics → Prometheus → Grafana

Alerting: Grafana Alerts / PagerDuty / OpsGenie
  → "Payment service p99 latency > 2s for 5 minutes" → alert on-call team
```

---

## CQRS and Event Sourcing

### CQRS — Command Query Responsibility Segregation

**Simple idea:** use a **different model for writing data** than for reading data.

```
Traditional approach (same model for read and write):
  POST /orders     → OrderService writes to Orders DB
  GET  /orders/123 → OrderService reads from Orders DB
  (same DB, same model — works fine for simple apps)

CQRS approach (separate models):
  POST /orders  → Command side → writes to Write DB (normalized, optimized for writes)
                → publishes OrderCreated event
                → Read side consumes event → updates Read DB (denormalized, optimized for queries)
  GET /orders/123 → Query side → reads from Read DB (fast, pre-joined data)
```

**Why?**
- Read traffic is usually 10-100x more than write traffic — scale them independently
- Read model can be a different DB type (e.g., write to PostgreSQL, read from Elasticsearch)
- Complex reports/dashboards don't slow down the write path

**Trade-off:** Eventual consistency — the read model may lag behind the write model by milliseconds.

---

### Event Sourcing

**Simple idea:** instead of storing the *current state*, store the *history of events* that led to it.

```
Traditional (store current state):
  Orders table: { id: 1, status: "SHIPPED", amount: 100 }

Event Sourcing (store events):
  Events table:
    { orderId: 1, event: "OrderCreated",  amount: 100, time: T1 }
    { orderId: 1, event: "PaymentDone",   time: T2 }
    { orderId: 1, event: "OrderShipped",  time: T3 }

Current state = replay all events
```

**Benefits:**
- Full audit trail — know exactly what happened and when
- Replay events to rebuild state or create new read models
- Time travel — see what the state was at any point in time
- Natural fit with SAGA pattern (each step is an event)

**Tools:** Axon Framework, EventStoreDB, Kafka (as event log)

---

### CQRS + Event Sourcing Together

```
Command Side                    Event Store              Query Side
────────────                    ───────────              ──────────
PlaceOrder command              OrderCreated event  →    Order read model (PostgreSQL)
  → validate                    PaymentDone event   →    Dashboard read model (Elasticsearch)
  → save events                 OrderShipped event  →    Notification read model (Redis)
  → publish to Kafka
```

Common in banking, e-commerce, and any domain needing full audit trails.

---

## Security in Microservices

### The Problem

In a monolith, one security layer protects everything. In microservices:
- Requests cross network boundaries between services
- Any compromised service could call any other service freely
- External JWT tokens must be validated at every service

---

### 1. External Security — JWT / OAuth2

```
User → API Gateway → validates JWT token
                   → extracts user info (userId, roles)
                   → passes claims in headers to downstream services
                   → downstream services trust the gateway
```

Flow:
```
1. User logs in → Auth Service issues JWT token
2. User sends JWT in every request: Authorization: Bearer <token>
3. API Gateway validates JWT (signature, expiry)
4. Gateway forwards request with user context header
5. Microservices read user context — no need to re-validate JWT
```

Tools: **Keycloak, Auth0, AWS Cognito, Okta**

---

### 2. Internal Security — mTLS (Mutual TLS)

Problem: Service A can freely call Service B — a compromised service can call anything.

mTLS solution: **both sides prove their identity** with certificates:

```
Without mTLS:
  Order Service → Payment Service  (anyone can call Payment Service)

With mTLS:
  Order Service presents certificate → Payment Service verifies it
  Payment Service presents certificate → Order Service verifies it
  Both sides authenticated — man-in-the-middle attack impossible
```

In Kubernetes, **Istio** handles mTLS automatically with zero app code changes — it injects a sidecar proxy into every pod that handles all TLS handshakes.

---

### 3. Zero-Trust Security Model

**Old model (castle and moat):** trust everything inside the network, block everything outside.

**Zero-trust:** trust nothing, verify everything — even internal service-to-service calls.

```
Principles:
  1. Every request must be authenticated (mTLS between services)
  2. Every request must be authorized (service A is allowed to call service B)
  3. Least privilege — services only have permissions they need
  4. Assume breach — design as if attackers are already inside
```

Enterprise implementation:
```
Istio Service Mesh
  ├── mTLS between all pods (automatic)
  ├── AuthorizationPolicy — "only Order Service can call Payment Service"
  ├── JWT validation at the mesh level
  └── Audit logging of all service-to-service calls
```

---

### 4. Secrets Management

Never hardcode passwords, API keys, or DB credentials in code or ConfigMaps:

```
❌ Wrong:
  application.properties: db.password=mysecretpassword

✅ Right:
  Kubernetes Secrets (base64 encoded, restricted RBAC)
  HashiCorp Vault (dynamic secrets, auto-rotation)
  AWS Secrets Manager / Azure Key Vault
```

---

## API Versioning in Microservices

### Why It's Hard

Services are consumed by other services and external clients. When you change an API:
- Old consumers break if you remove/rename fields
- You can't redeploy all consumers simultaneously
- Must support multiple versions in parallel

---

### Versioning Strategies

#### 1. URI Versioning (Most Common)
```
GET /v1/orders/123   ← old version still works
GET /v2/orders/123   ← new version with new response structure
```
Simple, explicit, easy to route at API Gateway level.

#### 2. Header Versioning
```
GET /orders/123
Accept: application/vnd.myapp.v2+json
```
Cleaner URLs but harder to test in browser.

#### 3. Query Parameter Versioning
```
GET /orders/123?version=2
```
Easy but pollutes query params.

---

### Best Practices

```
1. Never break backward compatibility — only add fields, never remove/rename
2. Deprecate old versions with a sunset date, not remove immediately
3. Support at least 2 versions simultaneously during transition
4. Use API Gateway to route versions to different service instances
5. Consumer-driven contract testing (Pact) to catch breaking changes early
```

---

## Testing Strategies in Microservices

### The Testing Pyramid

```
           /\
          /  \       E2E Tests (few, slow, expensive)
         /────\      ← test full user journey across services
        /      \
       /────────\    Integration Tests (moderate)
      /          \   ← test service + its DB, test service-to-service via stubs
     /────────────\
    /              \  Unit Tests (many, fast, cheap)
   /────────────────\ ← test business logic in isolation
```

---

### Types of Tests for Microservices

#### 1. Unit Tests
Test business logic in isolation — mock all dependencies:
```java
@Test
void shouldCalculateOrderTotal() {
    OrderService service = new OrderService(mockRepo, mockPaymentClient);
    Order order = service.createOrder(items);
    assertEquals(150.0, order.getTotal());
}
```

#### 2. Integration Tests
Test the service with its real database, but mock other services:
```java
@SpringBootTest
@Testcontainers  // spins up real PostgreSQL in Docker
void shouldSaveOrderToDatabase() { ... }
```

#### 3. Contract Testing (Most Important for Microservices)

**Problem:** Service A calls Service B — how do you make sure B doesn't break A's expectations?

**Solution: Consumer-Driven Contract Testing with Pact**

```
Consumer (Order Service) defines what it expects from Payment Service:
  "I expect POST /payments to return { transactionId, status }"

Pact saves this as a contract file.

Provider (Payment Service) verifies it runs the contract:
  "Does our actual API match what Order Service expects?"
  → Pass ✅ or Fail ❌ before deployment
```

This catches breaking API changes **without running both services together**.

#### 4. End-to-End Tests
Test the full user journey through all services — run sparingly, too slow and flaky for every commit.

---

## Deployment Patterns

### 1. Blue-Green Deployment

Run two identical environments — switch traffic instantly:

```
Current state:
  Blue environment (v1) ← 100% traffic
  Green environment (v2) ← 0% traffic (new version deployed here)

After validation:
  Blue environment (v1) ← 0% traffic
  Green environment (v2) ← 100% traffic (instant switch at Load Balancer)

Rollback: switch traffic back to Blue instantly
```

**Benefits:** zero downtime, instant rollback
**Cost:** double infrastructure during deployment

---

### 2. Canary Release

Gradually shift traffic to new version:

```
Day 1:  v1 = 95% traffic,  v2 = 5%  (canary — watch for errors)
Day 2:  v1 = 80% traffic,  v2 = 20%
Day 3:  v1 = 50% traffic,  v2 = 50%
Day 5:  v1 = 0%  traffic,  v2 = 100% (full rollout)

If v2 shows errors at any stage → route 100% back to v1
```

**Benefits:** real-world validation with limited blast radius
**Tools:** Istio traffic splitting, Kubernetes Argo Rollouts, AWS CodeDeploy

---

### 3. Feature Flags

Deploy code but control who sees the feature via configuration, not deployment:

```java
if (featureFlags.isEnabled("new-payment-flow", userId)) {
    return newPaymentService.process(order);
} else {
    return oldPaymentService.process(order);
}
```

**Benefits:**
- Deploy anytime, release when ready
- A/B testing — show new feature to 10% of users
- Kill switch — turn off a broken feature without redeployment

**Tools:** LaunchDarkly, Unleash, AWS AppConfig

---

### 4. Rolling Deployment (Kubernetes Default)

Already covered — K8s replaces pods one by one during updates.

---

## Strangler Fig Pattern — Monolith to Microservices Migration

### The Problem

You have a big monolith. You can't rewrite it overnight — that's "big bang rewrite" and it almost always fails.

### The Strategy — Strangle it Gradually

Named after the strangler fig tree — it grows around an existing tree, slowly replacing it until the original tree is gone.

```
Phase 1: Put a facade (API Gateway) in front of the monolith
  All traffic → API Gateway → Monolith (unchanged)

Phase 2: Extract one service at a time
  /payments → API Gateway → Payment Microservice (new)
  /orders   → API Gateway → Monolith (still there)
  /users    → API Gateway → Monolith (still there)

Phase 3: Extract more services over time
  /payments   → Payment Microservice
  /orders     → Order Microservice
  /inventory  → Inventory Microservice
  /users      → Monolith (last piece)

Phase 4: Monolith is fully replaced
  All routes → Microservices
  Monolith decommissioned
```

### Key Principles

- Never stop the business to do a rewrite — strangle while the monolith runs
- Extract high-value, high-change services first (payments, user auth)
- Share the database initially, then gradually split it per service
- Use the Anti-Corruption Layer to translate between monolith's data model and the new service's model

---

## CAP Theorem

### What It Says

In a distributed system, you can only guarantee **two of these three** at the same time:

```
C — Consistency:    every read gets the latest write
A — Availability:   every request gets a response (no timeout)
P — Partition tolerance: system works even if network between nodes fails
```

**Network partitions ALWAYS happen** in distributed systems — so you must choose between **C** and **A** when a partition occurs.

---

### CA vs CP vs AP Systems

| Type | Guarantees | Example | Trade-off |
|---|---|---|---|
| **CP** | Consistent + Partition tolerant | ZooKeeper, HBase, MongoDB (strong) | May return error/timeout instead of stale data |
| **AP** | Available + Partition tolerant | Cassandra, DynamoDB, CouchDB | May return stale/inconsistent data |
| **CA** | Consistent + Available | Traditional RDBMS (single node) | Not partition tolerant — can't scale horizontally |

---

### In Microservices Context

Each service picks the right DB for its needs:

```
Payment Service  → PostgreSQL (CP) — consistency critical, never show wrong balance
Product Catalog  → Cassandra (AP) — ok if price shows $99 for 1 second instead of $98
Session Store    → Redis (AP)     — ok if session data is slightly stale
Order Search     → Elasticsearch (AP) — ok if new order appears in search after 1s
```

**Key interview point:** microservices embrace **eventual consistency** (AP) for most services — the system will become consistent eventually, just not immediately.

---

## Idempotency in Microservices

### The Problem

In distributed systems, networks are unreliable. A request may:
- Get sent but the response is lost → caller retries → **duplicate processing**
- Payment charged twice, order created twice

```
Order Service → POST /payments → (network blip) → no response
Order Service retries → POST /payments → Payment charged AGAIN ❌
```

### Solution — Idempotency Keys

Every mutating request carries a unique **Idempotency Key**:

```
POST /payments
Idempotency-Key: order-456-attempt-1

Payment Service:
  1. Check if Idempotency-Key already processed
  2. If yes → return the same response as before (don't charge again)
  3. If no  → process payment, save key + result in DB
```

**Result:** retry the same request 10 times → payment charged exactly once ✅

---

### At-Least-Once vs Exactly-Once Delivery

| Delivery Type | Meaning | Risk | Used By |
|---|---|---|---|
| **At-most-once** | Message sent once, may be lost | Data loss | Fire-and-forget logs |
| **At-least-once** | Message always delivered, may be duplicated | Duplicate processing | Kafka default, most message brokers |
| **Exactly-once** | Delivered once and only once | Complex, slow | Kafka transactions, critical financial systems |

**Practical approach:** use **at-least-once** delivery + **idempotent consumers** (handle duplicates gracefully) — this is simpler and more performant than exactly-once.

---

## Microservice Anti-Patterns

### 1. Nano-Services (Too Fine-Grained)
```
❌ Wrong:
  GetUserNameService
  GetUserEmailService
  GetUserAgeService
  (one service per field!)

✅ Right:
  UserService (owns all user data)
```
Each service call is a network hop — too granular = massive latency and operational overhead.

---

### 2. Shared Database Anti-Pattern
```
❌ Wrong:
  Order Service ──┐
  Payment Service─┼──→ shared_database
  User Service ───┘

One schema change breaks all 3 services.
Teams block each other on DB migrations.
Services are tightly coupled despite being "micro".
```

---

### 3. Chatty Services
```
❌ Wrong:
  To display an order page, API Gateway makes:
  GET /users/123
  GET /orders/456
  GET /payments/789
  GET /inventory/101
  GET /shipping/202
  (5 round trips, 5x network latency)

✅ Right:
  Backend For Frontend (BFF) pattern or GraphQL
  One aggregated call that fetches everything needed
```

---

### 4. Synchronous Chain of Calls
```
❌ Wrong:
  Gateway → Order → Payment → Inventory → Notification
  (synchronous chain — if Notification is slow, everything waits)

✅ Right:
  Order Service handles the request, publishes an event
  Payment, Inventory, Notification react asynchronously
```

---

### 5. Ignoring Backward Compatibility
```
❌ Wrong:
  Payment Service renames field "amount" to "totalAmount"
  → Order Service breaks immediately
  → Every consumer breaks without warning

✅ Right:
  Add new field, keep old field (deprecate with sunset date)
  Use consumer-driven contract tests to catch breaking changes
```

---

## gRPC vs REST vs GraphQL in Microservices

### Quick Comparison

| | REST | gRPC | GraphQL |
|---|---|---|---|
| **Protocol** | HTTP/1.1 | HTTP/2 | HTTP/1.1 or HTTP/2 |
| **Format** | JSON (text) | Protocol Buffers (binary) | JSON |
| **Speed** | Moderate | Fast (5-10x faster than REST) | Moderate |
| **Contract** | OpenAPI/Swagger (optional) | `.proto` file (mandatory, strict) | Schema (mandatory) |
| **Streaming** | No (workarounds exist) | Yes (bidirectional) | Subscriptions |
| **Browser support** | Full | Limited (needs grpc-web) | Full |
| **Best for** | External APIs, public APIs | Internal service-to-service | Client-driven queries, BFF layer |

---

### When to Use Which

```
External API (mobile app, third party developers)
  → REST — universal support, easy to document, familiar

Internal service-to-service (high volume, low latency)
  → gRPC — faster, strict contracts, streaming support
  → e.g., Order Service → Inventory Service (millions of calls/day)

Frontend / BFF (client needs flexible data)
  → GraphQL — client asks for exactly what it needs, no over/under-fetching
  → e.g., dashboard that aggregates data from 5 services in one query
```

---

### gRPC Key Concepts

```protobuf
// payment.proto — strict contract, shared between services
service PaymentService {
  rpc ProcessPayment (PaymentRequest) returns (PaymentResponse);
  rpc StreamTransactions (UserRequest) returns (stream Transaction); // streaming!
}

message PaymentRequest {
  string order_id = 1;
  double amount = 2;
}
```

- `.proto` file is the single source of truth — breaking changes caught at compile time
- Binary format = smaller payload, faster serialization than JSON

---

## Configuration Management in Microservices

### The Problem

Hardcoding config in each service:
- Different config needed per environment (dev/staging/prod)
- Secrets (DB passwords, API keys) must not be in code
- Config change requires redeployment

---

### Solutions

#### 1. Kubernetes ConfigMaps (Non-sensitive config)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  DATABASE_URL: "jdbc:postgresql://order-db:5432/orders"
  MAX_RETRY_COUNT: "3"
  FEATURE_FLAG_NEW_CHECKOUT: "true"
```
Injected as environment variables into pods — no code change needed.

#### 2. Kubernetes Secrets (Sensitive config)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
data:
  DB_PASSWORD: bXlwYXNzd29yZA==  # base64 encoded
  API_KEY: c2VjcmV0a2V5MTIz
```
Access restricted via RBAC — only pods that need them can mount them.

#### 3. Spring Cloud Config Server
Centralized config server backed by a Git repo:
```
Git Repo (configs)
    │
Spring Cloud Config Server
    │
    ├── Order Service   → fetches config on startup
    ├── Payment Service → fetches config on startup
    └── User Service    → fetches config on startup

Config change → commit to Git → services refresh config without restart
```

#### 4. HashiCorp Vault (Enterprise Secret Management)
```
Features:
  - Dynamic secrets — generates DB credentials on the fly, auto-expires
  - Secret rotation — rotates API keys automatically
  - Fine-grained access control — service A can only read its own secrets
  - Full audit log of every secret access
```

---

### Best Practices

| Rule | Reason |
|---|---|
| Never commit secrets to Git | Leaked credentials are the #1 security breach cause |
| Externalize all config | 12-factor app principle — same image, different config per env |
| Use Vault for secrets in production | K8s Secrets are only base64, not encrypted at rest by default |
| Refresh config without restart | Use Spring Cloud Config + `@RefreshScope` or K8s ConfigMap reloading |
