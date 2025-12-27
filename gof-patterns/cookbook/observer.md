# Observer Pattern - Jakarta EE Cookbook

## Intent

Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

## Jakarta EE Implementation

CDI provides built-in Observer pattern through `Event<T>` and `@Observes`.

### 1. Event Definition

```java
// Event payload
public class OrderCreatedEvent {
    private final Order order;
    private final Instant timestamp;

    public OrderCreatedEvent(Order order) {
        this.order = order;
        this.timestamp = Instant.now();
    }

    public Order getOrder() { return order; }
    public Instant getTimestamp() { return timestamp; }
}

public class OrderCancelledEvent {
    private final Order order;
    private final String reason;

    public OrderCancelledEvent(Order order, String reason) {
        this.order = order;
        this.reason = reason;
    }

    // getters
}
```

### 2. Event Producer

```java
@ApplicationScoped
public class OrderService {

    @Inject
    Event<OrderCreatedEvent> orderCreatedEvent;

    @Inject
    Event<OrderCancelledEvent> orderCancelledEvent;

    @Inject
    OrderRepository orders;

    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        orders.save(order);

        // Fire event - all observers notified
        orderCreatedEvent.fire(new OrderCreatedEvent(order));

        return order;
    }

    @Transactional
    public Order cancelOrder(Long orderId, String reason) {
        Order order = orders.findById(orderId);
        order.setStatus(OrderStatus.CANCELLED);
        orders.save(order);

        orderCancelledEvent.fire(new OrderCancelledEvent(order, reason));

        return order;
    }
}
```

### 3. Synchronous Observers

```java
@ApplicationScoped
public class InventoryObserver {

    @Inject
    InventoryService inventory;

    // Synchronous - runs in same transaction
    public void onOrderCreated(@Observes OrderCreatedEvent event) {
        inventory.reserve(event.getOrder().getItems());
    }

    public void onOrderCancelled(@Observes OrderCancelledEvent event) {
        inventory.release(event.getOrder().getItems());
    }
}

@ApplicationScoped
public class EmailObserver {

    @Inject
    EmailService email;

    public void onOrderCreated(@Observes OrderCreatedEvent event) {
        email.sendOrderConfirmation(event.getOrder());
    }
}
```

### 4. Asynchronous Observers

```java
@ApplicationScoped
public class AnalyticsObserver {

    @Inject
    AnalyticsService analytics;

    // Async - runs in separate thread, outside transaction
    public void onOrderCreated(@ObservesAsync OrderCreatedEvent event) {
        analytics.trackOrderCreation(event.getOrder());
    }
}

// Producer for async events
@ApplicationScoped
public class OrderService {

    @Inject
    Event<OrderCreatedEvent> orderCreatedEvent;

    public Order createOrder(OrderRequest request) {
        Order order = saveOrder(request);

        // Fire async - returns immediately
        orderCreatedEvent.fireAsync(new OrderCreatedEvent(order));

        return order;
    }
}
```

### 5. Conditional Observers with Qualifiers

```java
@Qualifier
@Retention(RUNTIME)
@Target({TYPE, METHOD, FIELD, PARAMETER})
public @interface Priority {
    Level value();

    enum Level { HIGH, NORMAL, LOW }
}

// Qualified event firing
@Inject
@Priority(Priority.Level.HIGH)
Event<OrderCreatedEvent> highPriorityEvent;

// Qualified observer
public void onHighPriorityOrder(
        @Observes @Priority(Priority.Level.HIGH) OrderCreatedEvent event) {
    // Only receives high priority orders
}
```

### 6. Transaction-Aware Observers

```java
@ApplicationScoped
public class AuditObserver {

    // Runs only if transaction succeeds
    public void onOrderCreatedSuccess(
            @Observes(during = TransactionPhase.AFTER_SUCCESS)
            OrderCreatedEvent event) {
        auditLog.record("ORDER_CREATED", event.getOrder());
    }

    // Runs only if transaction fails
    public void onOrderCreatedFailure(
            @Observes(during = TransactionPhase.AFTER_FAILURE)
            OrderCreatedEvent event) {
        alertService.notify("Order creation failed: " + event.getOrder().getId());
    }
}
```

## When to Use

✅ **Use Observer when:**

- Multiple objects need to react to state changes
- You want to decouple event producers from consumers
- You need to add new listeners without modifying the producer

❌ **Avoid Observer when:**

- Simple one-to-one communication (direct call is clearer)
- Order of observer execution matters (use explicit orchestration)
