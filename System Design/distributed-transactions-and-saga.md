# Distributed Transactions and SAGA Pattern in Microservices

A comprehensive guide to understanding distributed transactions, two-phase commit (2PC), and the SAGA pattern in event-driven microservices.

---

## Table of Contents

1. [The Problem: Distributed Transactions](#the-problem-distributed-transactions)
2. [Solution 1: Two-Phase Commit (2PC)](#solution-1-two-phase-commit-2pc)
3. [Solution 2: SAGA Pattern](#solution-2-saga-pattern)
4. [SAGA Choreography vs Orchestration](#saga-choreography-vs-orchestration)
5. [Real-World Example](#real-world-example-food-delivery)
6. [2PC vs SAGA Comparison](#key-differences-2pc-vs-saga)
7. [SAGA Design Considerations](#saga-design-considerations)

---

## The Problem: Distributed Transactions

In microservices, a **distributed transaction** spans multiple services that each have their own database. The challenge: how do you ensure **all services commit or all rollback** when they're independent?

### Example: E-commerce Order

When a user places an order, multiple things must happen **atomically**:

```
1. OrderService    → Create order record
2. PaymentService  → Charge credit card
3. InventoryService → Reserve stock
4. ShippingService → Create shipment
```

**What if Payment succeeds but Inventory fails?** You've charged the customer but can't fulfill the order. 💥

### The Monolithic Solution

In a monolithic app with one database, you'd use a **database transaction** (ACID):

```sql
BEGIN TRANSACTION;
  INSERT INTO orders ...;
  INSERT INTO payments ...;
  UPDATE inventory ...;
COMMIT;
```

**But in microservices**, each service has its own database. You can't use a single DB transaction across them.

---

## Solution 1: Two-Phase Commit (2PC)

### How 2PC Works

A **coordinator** manages the transaction across all services in two phases:

```
Phase 1: PREPARE
  Coordinator → "Can you commit?"
  OrderService    → "Yes, ready"
  PaymentService  → "Yes, ready"
  InventoryService → "Yes, ready"

Phase 2: COMMIT
  Coordinator → "Everyone commit now!"
  All services → Execute and lock in changes
```

If **any service says "No"** in Phase 1, the coordinator tells everyone to **ROLLBACK**.

### Visual Flow

```
Coordinator
    │
    ├──► OrderService:    "Prepare?" → "Ready ✓"
    ├──► PaymentService:  "Prepare?" → "Ready ✓"
    └──► InventoryService: "Prepare?" → "Ready ✓"
    
    ↓ (All ready)
    
    ├──► OrderService:    "Commit!" → Done
    ├──► PaymentService:  "Commit!" → Done
    └──► InventoryService: "Commit!" → Done
```

### Detailed Example: Order Processing

#### Phase 1: Prepare

```
1. Coordinator sends PREPARE to all services

2. OrderService:
   - Validates order data
   - Locks order record
   - Returns: "READY"

3. PaymentService:
   - Validates credit card
   - Holds the amount
   - Returns: "READY"

4. InventoryService:
   - Checks stock availability
   - Locks inventory items
   - Returns: "READY"
```

#### Phase 2: Commit

```
5. Coordinator receives all "READY" responses

6. Coordinator sends COMMIT to all services

7. OrderService:
   - Persists order
   - Releases lock
   - Returns: "COMMITTED"

8. PaymentService:
   - Charges the card
   - Releases hold
   - Returns: "COMMITTED"

9. InventoryService:
   - Reduces stock
   - Releases lock
   - Returns: "COMMITTED"
```

#### Rollback Scenario

```
If InventoryService returns "NOT READY" (out of stock):

1. Coordinator sends ROLLBACK to all services

2. OrderService:
   - Discards order
   - Releases lock

3. PaymentService:
   - Releases hold on card
   - No charge made
```

### Problems with 2PC

❌ **Blocking** — services lock resources while waiting for coordinator  
❌ **Single point of failure** — if coordinator crashes, everyone is stuck  
❌ **Slow** — synchronous, waits for slowest service  
❌ **Not cloud-native** — doesn't handle network partitions well  
❌ **Reduced availability** — all services must be up for transaction to succeed

**Reality:** 2PC is rarely used in modern microservices because it kills the benefits of decoupling.

---

## Solution 2: SAGA Pattern

Instead of a single atomic transaction, SAGA breaks it into **multiple local transactions**, each publishing an event. If something fails, **compensating transactions** undo previous steps.

### Core Concept

> A SAGA is a sequence of local transactions where each transaction updates data within a single service and publishes an event or message to trigger the next transaction step.

If a step fails, the SAGA executes **compensating transactions** to undo the changes made by preceding steps.

### Two Types of SAGA

| Type | Coordination |
|---|---|
| **Choreography** | No central coordinator; services react to events |
| **Orchestration** | Central orchestrator tells each service what to do |

---

## SAGA: Choreography

### How It Works

Services communicate by publishing and subscribing to events. Each service:
1. Performs its local transaction
2. Publishes an event
3. Listens for events from other services

### Happy Path (Success)

```
1. OrderService
   ├─ Create order (status: PENDING)
   └─ Publish: "OrderCreated"
   
2. PaymentService (listens to "OrderCreated")
   ├─ Charge card
   └─ Publish: "PaymentCompleted"
   
3. InventoryService (listens to "PaymentCompleted")
   ├─ Reserve stock
   └─ Publish: "InventoryReserved"
   
4. ShippingService (listens to "InventoryReserved")
   ├─ Create shipment
   └─ Publish: "ShipmentCreated"
   
5. OrderService (listens to "ShipmentCreated")
   └─ Update order (status: CONFIRMED)
```

Each service does its local transaction and publishes an event. The next service reacts.

### Failure Path (Compensation)

What if **Inventory fails** (out of stock)?

```
1. OrderService
   └─ Publish: "OrderCreated"
   
2. PaymentService
   └─ Publish: "PaymentCompleted" (card charged ✓)
   
3. InventoryService
   ├─ Check stock → OUT OF STOCK ❌
   └─ Publish: "InventoryReservationFailed"
   
4. PaymentService (listens to "InventoryReservationFailed")
   ├─ REFUND the card (compensating transaction)
   └─ Publish: "PaymentRefunded"
   
5. OrderService (listens to "PaymentRefunded")
   └─ Update order (status: CANCELLED)
```

**Compensating transactions** undo what was done. The payment was charged, so we refund it.

### Code Example: Choreography

**OrderService:**
```javascript
// Create order and publish event
async function createOrder(orderData) {
  const order = await db.orders.create({
    ...orderData,
    status: 'PENDING'
  });
  
  await eventBus.publish('OrderCreated', {
    orderId: order.id,
    userId: order.userId,
    amount: order.amount
  });
  
  return order;
}

// Listen for completion or failure
eventBus.subscribe('ShipmentCreated', async (event) => {
  await db.orders.update(event.orderId, { status: 'CONFIRMED' });
});

eventBus.subscribe('PaymentRefunded', async (event) => {
  await db.orders.update(event.orderId, { status: 'CANCELLED' });
});
```

**PaymentService:**
```javascript
// Listen for order creation
eventBus.subscribe('OrderCreated', async (event) => {
  try {
    const payment = await chargeCard(event.userId, event.amount);
    
    await eventBus.publish('PaymentCompleted', {
      orderId: event.orderId,
      paymentId: payment.id
    });
  } catch (error) {
    await eventBus.publish('PaymentFailed', {
      orderId: event.orderId,
      reason: error.message
    });
  }
});

// Listen for inventory failure to compensate
eventBus.subscribe('InventoryReservationFailed', async (event) => {
  await refundCard(event.paymentId);
  
  await eventBus.publish('PaymentRefunded', {
    orderId: event.orderId
  });
});
```

---

## SAGA: Orchestration

### How It Works

A **central orchestrator** (e.g., `OrderOrchestrator`) manages the workflow. It:
1. Sends commands to services
2. Receives responses
3. Decides next steps
4. Handles compensation on failure

### Happy Path

```
OrderOrchestrator receives "PlaceOrder" command:

1. Call PaymentService → "Charge card"
   ├─ Success → Continue
   
2. Call InventoryService → "Reserve stock"
   ├─ Success → Continue
   
3. Call ShippingService → "Create shipment"
   ├─ Success → Continue
   
4. Update OrderService → "Order confirmed"
```

### Failure Path

```
OrderOrchestrator receives "PlaceOrder" command:

1. Call PaymentService → "Charge card"
   ├─ Success ✓
   
2. Call InventoryService → "Reserve stock"
   ├─ FAILED (out of stock) ❌
   
3. Orchestrator triggers compensation:
   ├─ Call PaymentService → "Refund card"
   └─ Update OrderService → "Order cancelled"
```

The orchestrator knows the entire flow and handles rollback logic.

### Code Example: Orchestration

**OrderOrchestrator:**
```javascript
class OrderOrchestrator {
  async placeOrder(orderData) {
    const saga = new SagaInstance(orderData);
    
    try {
      // Step 1: Create order
      const order = await this.orderService.createOrder(orderData);
      saga.addCompensation(() => this.orderService.cancelOrder(order.id));
      
      // Step 2: Charge payment
      const payment = await this.paymentService.charge({
        userId: orderData.userId,
        amount: orderData.amount
      });
      saga.addCompensation(() => this.paymentService.refund(payment.id));
      
      // Step 3: Reserve inventory
      const reservation = await this.inventoryService.reserve({
        orderId: order.id,
        items: orderData.items
      });
      saga.addCompensation(() => this.inventoryService.release(reservation.id));
      
      // Step 4: Create shipment
      const shipment = await this.shippingService.createShipment({
        orderId: order.id,
        address: orderData.address
      });
      
      // Success - confirm order
      await this.orderService.confirmOrder(order.id);
      
      return { success: true, orderId: order.id };
      
    } catch (error) {
      // Failure - run compensations in reverse order
      await saga.compensate();
      
      return { success: false, error: error.message };
    }
  }
}

class SagaInstance {
  constructor(data) {
    this.data = data;
    this.compensations = [];
  }
  
  addCompensation(fn) {
    this.compensations.push(fn);
  }
  
  async compensate() {
    // Execute compensations in reverse order
    for (const compensation of this.compensations.reverse()) {
      try {
        await compensation();
      } catch (error) {
        console.error('Compensation failed:', error);
      }
    }
  }
}
```

---

## SAGA: Choreography vs Orchestration

| | **Choreography** | **Orchestration** |
|---|---|---|
| **Coordination** | Decentralized (events) | Centralized (orchestrator) |
| **Coupling** | Loose | Tighter (services know orchestrator) |
| **Complexity** | Hard to trace flow | Easier to understand flow |
| **Failure handling** | Each service handles compensation | Orchestrator handles compensation |
| **Cyclic dependencies** | Risk of circular event chains | No risk (orchestrator controls flow) |
| **Testing** | Harder (need to test event chains) | Easier (test orchestrator logic) |
| **Scalability** | Better (no central bottleneck) | Orchestrator can be bottleneck |
| **Best for** | Simple workflows, few services | Complex workflows, many steps |

### When to Use Each

**Use Choreography when:**
- Simple workflows (2-4 steps)
- Services are truly independent
- You want maximum decoupling
- Event-driven architecture is already in place

**Use Orchestration when:**
- Complex workflows (5+ steps)
- Need clear visibility of business process
- Conditional logic between steps
- Easier debugging is important

---

## Real-World Example: Food Delivery

### Scenario: User orders food

**Services involved:**
- OrderService
- PaymentService
- RestaurantService
- DeliveryService

### Choreography SAGA

```
1. OrderService
   └─ Publish: "OrderPlaced" { orderId, userId, restaurantId, amount }

2. PaymentService (subscribes to "OrderPlaced")
   ├─ Charge user
   └─ Publish: "PaymentSucceeded" OR "PaymentFailed"

3. RestaurantService (subscribes to "PaymentSucceeded")
   ├─ Accept order
   └─ Publish: "OrderAccepted" OR "OrderRejected" (restaurant closed)

4. DeliveryService (subscribes to "OrderAccepted")
   ├─ Assign driver
   └─ Publish: "DriverAssigned" OR "NoDriverAvailable"

If "OrderRejected":
   → PaymentService refunds
   → OrderService marks order as cancelled

If "NoDriverAvailable":
   → RestaurantService cancels preparation
   → PaymentService refunds
   → OrderService marks order as cancelled
```

### Orchestration SAGA

```javascript
class FoodDeliveryOrchestrator {
  async processOrder(orderRequest) {
    const saga = new SagaInstance();
    
    try {
      // 1. Create order
      const order = await orderService.create(orderRequest);
      saga.addCompensation(() => orderService.cancel(order.id));
      
      // 2. Charge payment
      const payment = await paymentService.charge(order.userId, order.amount);
      saga.addCompensation(() => paymentService.refund(payment.id));
      
      // 3. Notify restaurant
      const acceptance = await restaurantService.acceptOrder(order.id);
      if (!acceptance.accepted) {
        throw new Error('Restaurant rejected order');
      }
      saga.addCompensation(() => restaurantService.cancelOrder(order.id));
      
      // 4. Assign delivery driver
      const driver = await deliveryService.assignDriver(order.id);
      if (!driver) {
        throw new Error('No driver available');
      }
      
      // Success
      await orderService.confirm(order.id);
      return { success: true, orderId: order.id, driver: driver.id };
      
    } catch (error) {
      await saga.compensate();
      return { success: false, error: error.message };
    }
  }
}
```

---

## Key Differences: 2PC vs SAGA

| | **2PC** | **SAGA** |
|---|---|---|
| **Consistency** | Strong (ACID) | Eventual |
| **Isolation** | Full (locks resources) | None (dirty reads possible) |
| **Locking** | Yes (blocks resources) | No |
| **Performance** | Slow (synchronous) | Fast (asynchronous) |
| **Availability** | Low (all must be up) | High (services independent) |
| **Failure handling** | Automatic rollback | Compensating transactions |
| **Complexity** | Simple concept, hard ops | Complex logic, easier ops |
| **Scalability** | Poor | Excellent |
| **Use in microservices** | Rare (anti-pattern) | Common (best practice) |

### Why SAGA Over 2PC?

**2PC Problems:**
```
Service A ──┐
Service B ──┼── Coordinator (SPOF)
Service C ──┘
     ↓
All blocked waiting for slowest service
If coordinator crashes, all stuck in limbo
```

**SAGA Benefits:**
```
Service A → Event → Service B → Event → Service C
     ↓           ↓           ↓
Each proceeds independently
Failures handled via compensation
No blocking, no SPOF
```

---

## SAGA Design Considerations

### 1. Idempotency

Services must handle duplicate events safely:

```javascript
// BAD: Not idempotent
eventBus.subscribe('PaymentCompleted', async (event) => {
  await chargeCard(event.userId, event.amount); // Charges twice if event replayed!
});

// GOOD: Idempotent
eventBus.subscribe('PaymentCompleted', async (event) => {
  const existing = await db.payments.findOne({ eventId: event.id });
  if (existing) {
    return; // Already processed
  }
  
  await chargeCard(event.userId, event.amount);
  await db.payments.create({ eventId: event.id, ...event });
});
```

### 2. Compensating Transactions

Not all actions are reversible:

| Action | Compensation |
|---|---|
| ✅ Charge payment | Refund payment |
| ✅ Reserve inventory | Release inventory |
| ✅ Lock resource | Unlock resource |
| ❌ Send email | ~~Un-send email~~ → Send cancellation email |
| ❌ Print shipping label | ~~Un-print~~ → Void label |
| ❌ External API call | Depends on API (may not be reversible) |

**Design compensations upfront** when designing the SAGA.

### 3. Eventual Consistency

The system is temporarily inconsistent:

```
Time 0: Payment charged ✓
Time 1: Inventory not yet reserved (brief window)
Time 2: Inventory reserved ✓
```

**User experience considerations:**
- Show "Order processing..." status
- Don't promise immediate confirmation
- Send email when order is fully confirmed

### 4. Observability and Tracing

You need distributed tracing to debug:

```
Trace ID: abc123
  OrderService (100ms) → PaymentService (200ms) → InventoryService (FAILED)
                                                         ↓
                                                   Compensation started
                                                         ↓
                                              PaymentService refund (150ms)
```

**Tools:**
- Jaeger
- Zipkin
- OpenTelemetry
- AWS X-Ray

### 5. Timeout Handling

What if a service never responds?

```javascript
class OrderOrchestrator {
  async placeOrder(orderData) {
    try {
      // Set timeout for each step
      const payment = await Promise.race([
        this.paymentService.charge(orderData),
        this.timeout(5000) // 5 second timeout
      ]);
      
      // Continue...
      
    } catch (error) {
      if (error.name === 'TimeoutError') {
        // Trigger compensation
        await this.compensate();
      }
    }
  }
  
  timeout(ms) {
    return new Promise((_, reject) => 
      setTimeout(() => reject(new Error('TimeoutError')), ms)
    );
  }
}
```

### 6. Semantic Lock

Prevent concurrent modifications:

```javascript
// Prevent user from placing multiple orders simultaneously
async function createOrder(userId, orderData) {
  const lock = await acquireLock(`user:${userId}:order`, ttl: 30000);
  
  try {
    // Check if user has pending order
    const pending = await db.orders.findOne({ 
      userId, 
      status: 'PENDING' 
    });
    
    if (pending) {
      throw new Error('Order already in progress');
    }
    
    // Create order
    return await db.orders.create(orderData);
    
  } finally {
    await lock.release();
  }
}
```

---

## Summary

| Concept | What It Is |
|---|---|
| **Distributed Transaction** | A transaction spanning multiple microservices with separate databases |
| **2PC (Two-Phase Commit)** | Synchronous protocol with prepare/commit phases; rarely used in microservices |
| **SAGA Pattern** | Sequence of local transactions with compensating transactions on failure |
| **Choreography SAGA** | Services react to events; no central coordinator |
| **Orchestration SAGA** | Central orchestrator manages the workflow |
| **Compensating Transaction** | Undo action (e.g., refund payment) when a step fails |
| **Idempotency** | Ability to handle duplicate events safely |
| **Eventual Consistency** | System reaches consistent state over time, not immediately |

### Decision Tree

```
Need distributed transaction?
    │
    ├─ Strong consistency required? → Use 2PC (rare)
    │
    └─ Eventual consistency OK? → Use SAGA
            │
            ├─ Simple workflow (2-4 steps)? → Choreography
            │
            └─ Complex workflow (5+ steps)? → Orchestration
```

**Bottom line:** In event-driven microservices, **SAGA is the standard** for handling distributed transactions. 2PC is avoided due to its blocking nature and poor fault tolerance.

---

## Additional Resources

- [Microservices Patterns: SAGA](https://microservices.io/patterns/data/saga.html)
- [Distributed Transactions: The Icebergs of Microservices](https://www.grahamlea.com/2016/08/distributed-transactions-microservices-icebergs/)
- [AWS: Saga Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-data-persistence/saga-pattern.html)
- [Martin Fowler: Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)

---

**Created:** April 2026  
**Purpose:** Educational guide for distributed transactions and SAGA pattern in microservices
