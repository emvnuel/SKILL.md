# Shotgun Surgery - Clean Code Cookbook

## Intent

Shotgun Surgery occurs when a single change requires modifying many different classes. This indicates that a responsibility is scattered across multiple classes and should be consolidated.

---

## The Problem

Shotgun Surgery means:

- One logical change touches many files
- Easy to miss a file and break functionality
- Hard to understand the full impact of a change
- Related behavior is scattered across the codebase

---

## Detection

You have Shotgun Surgery when:

- "Every time we change X, we have to update these 10 files"
- Searching for all usages of a concept returns many classes
- Features are implemented by modifying multiple scattered classes
- Changes frequently cause bugs from missed updates

---

## Before - Shotgun Surgery

Adding a new order status requires changes to many classes:

```java
// 1. Enum - add new status
public enum OrderStatus {
    PENDING,
    CONFIRMED,
    PROCESSING,
    SHIPPED,
    DELIVERED
    // ADD: RETURNED
}

// 2. Order entity - add transition validation
public class Order {
    public void updateStatus(OrderStatus newStatus) {
        // Add case for RETURNED
    }
}

// 3. REST resource - add endpoint
@Path("/orders")
public class OrderResource {
    // Add: @POST @Path("/{id}/return")
}

// 4. Service - add business logic
public class OrderService {
    // Add: public void returnOrder(OrderId id)
}

// 5. Email templates - add notification
// Add: returned-order.html

// 6. Email service - add method
public class EmailService {
    // Add: public void sendReturnConfirmation(Order order)
}

// 7. Database migration - add column/values
// Add: V20_add_returned_status.sql

// 8. Event publisher - add event
// Add: OrderReturnedEvent

// 9. Analytics - track new status
public class AnalyticsService {
    // Add: trackReturn(Order order)
}

// 10. UI - add button and view
// Modify multiple frontend files
```

---

## After - Consolidated Responsibility

### Use State Pattern

```java
// All status behavior in one place
public interface OrderState {

    OrderStatus getStatus();

    OrderState confirm(Order order);
    OrderState startProcessing(Order order);
    OrderState ship(Order order, TrackingNumber tracking);
    OrderState deliver(Order order);
    OrderState returnOrder(Order order, ReturnReason reason);

    // Each state knows what's valid
    default OrderState transition(Order order, OrderAction action) {
        return switch (action) {
            case CONFIRM -> confirm(order);
            case PROCESS -> startProcessing(order);
            case SHIP -> ship(order, null);
            case DELIVER -> deliver(order);
            case RETURN -> returnOrder(order, null);
        };
    }
}

@ApplicationScoped
@Named("pending")
public class PendingOrderState implements OrderState {

    @Override
    public OrderStatus getStatus() {
        return OrderStatus.PENDING;
    }

    @Override
    public OrderState confirm(Order order) {
        return new ConfirmedOrderState();
    }

    @Override
    public OrderState returnOrder(Order order, ReturnReason reason) {
        throw new InvalidStateTransitionException(this, "RETURN");
    }

    // ... other transitions throw for invalid actions
}

@ApplicationScoped
@Named("delivered")
public class DeliveredOrderState implements OrderState {

    @Override
    public OrderState returnOrder(Order order, ReturnReason reason) {
        // All return logic in one place
        order.setReturnReason(reason);
        return new ReturnedOrderState();
    }
}
```

### Central Order Lifecycle

```java
@ApplicationScoped
public class OrderLifecycleService {

    @Inject
    Event<OrderStatusChanged> statusEvents;

    @Inject
    OrderRepository repository;

    @Transactional
    public Order transitionTo(OrderId orderId, OrderAction action, Object... params) {
        Order order = repository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        OrderState oldState = order.getState();
        OrderState newState = oldState.transition(order, action);

        order.setState(newState);
        Order saved = repository.save(order);

        // Single event, listeners handle their concerns
        statusEvents.fire(new OrderStatusChanged(
            saved,
            oldState.getStatus(),
            newState.getStatus()
        ));

        return saved;
    }
}
```

### Event-Driven Side Effects

```java
// Notifications listen to one event
@ApplicationScoped
public class OrderNotificationListener {

    @Inject
    EmailService emailService;

    public void onStatusChanged(@Observes OrderStatusChanged event) {
        // All notification logic centralized
        switch (event.getNewStatus()) {
            case CONFIRMED -> emailService.sendConfirmation(event.getOrder());
            case SHIPPED -> emailService.sendShippedNotification(event.getOrder());
            case DELIVERED -> emailService.sendDeliveredNotification(event.getOrder());
            case RETURNED -> emailService.sendReturnConfirmation(event.getOrder());
        }
    }
}

// Analytics listen to same event
@ApplicationScoped
public class OrderAnalyticsListener {

    @Inject
    AnalyticsService analytics;

    public void onStatusChanged(@Observes OrderStatusChanged event) {
        analytics.trackStatusChange(
            event.getOrder().getId(),
            event.getOldStatus(),
            event.getNewStatus()
        );
    }
}
```

---

## Adding New Status Now

With the refactored design, adding RETURNED status:

```java
// 1. Add status enum value
public enum OrderStatus {
    // ... existing
    RETURNED
}

// 2. Add state class (all behavior in one place)
@ApplicationScoped
@Named("returned")
public class ReturnedOrderState implements OrderState {
    // All returned-order logic here
}

// 3. Update DeliveredOrderState to allow transition
// (Already done in the state class above)

// Done! Events propagate to listeners automatically
```

---

## Move Field / Move Method

When a field or method is used by many classes:

### Before - Field Accessed Everywhere

```java
public class Config {
    public static final int MAX_RETRY_ATTEMPTS = 3;
}

// Used in many places
public class PaymentService {
    public void process(Payment payment) {
        for (int i = 0; i < Config.MAX_RETRY_ATTEMPTS; i++) { ... }
    }
}

public class EmailService {
    public void send(Email email) {
        for (int i = 0; i < Config.MAX_RETRY_ATTEMPTS; i++) { ... }
    }
}

public class InventoryService {
    public void reserve(Item item) {
        for (int i = 0; i < Config.MAX_RETRY_ATTEMPTS; i++) { ... }
    }
}
```

### After - Behavior Consolidated

```java
@ApplicationScoped
public class RetryExecutor {

    @Inject
    @ConfigProperty(name = "app.retry.max-attempts", defaultValue = "3")
    int maxAttempts;

    public <T> T execute(Supplier<T> operation, String operationName) {
        Exception lastException = null;
        for (int attempt = 0; attempt < maxAttempts; attempt++) {
            try {
                return operation.get();
            } catch (RetryableException e) {
                lastException = e;
                log.warn("Attempt {} failed for {}", attempt + 1, operationName);
            }
        }
        throw new MaxRetriesExceededException(operationName, lastException);
    }
}

// Services delegate retry logic
@ApplicationScoped
public class PaymentService {

    @Inject
    RetryExecutor retryExecutor;

    public PaymentResult process(Payment payment) {
        return retryExecutor.execute(
            () -> paymentGateway.charge(payment),
            "process payment"
        );
    }
}
```

---

## Comparison with Divergent Change

| Shotgun Surgery                    | Divergent Change                  |
| ---------------------------------- | --------------------------------- |
| One change, many classes to modify | One class, many reasons to change |
| Consolidate into fewer classes     | Split into multiple classes       |
| Responsibility too scattered       | Too many responsibilities         |

---

## Refactoring Steps

1. **Identify the trigger**: What change causes shotgun surgery?
2. **Find all affected files**: List every file that changes together
3. **Extract common behavior**: Create class for the scattered logic
4. **Use events**: Decouple via publish/subscribe
5. **Move to single location**: Consolidate related code
6. **Inject dependency**: Other classes use the consolidated service

---

## Related Recipes

- [Divergent Change](./divergent-change.md): Opposite problem
- [Single Responsibility](./single-responsibility.md): Each class does one thing
- [Feature Envy](./feature-envy.md): Move behavior to right place
