# Mediator Pattern - Jakarta EE Cookbook

## Intent

Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly.

## Jakarta EE Implementation

### 1. Mediator Interface

```java
public interface OrderMediator {
    void notify(Object sender, String event, Object data);
}
```

### 2. Concrete Mediator

```java
@ApplicationScoped
public class OrderProcessingMediator implements OrderMediator {

    @Inject
    InventoryService inventory;

    @Inject
    PaymentService payment;

    @Inject
    ShippingService shipping;

    @Inject
    NotificationService notification;

    @Inject
    AnalyticsService analytics;

    @Override
    public void notify(Object sender, String event, Object data) {
        switch (event) {
            case "ORDER_CREATED" -> handleOrderCreated((Order) data);
            case "PAYMENT_COMPLETED" -> handlePaymentCompleted((PaymentResult) data);
            case "PAYMENT_FAILED" -> handlePaymentFailed((PaymentResult) data);
            case "ITEMS_SHIPPED" -> handleItemsShipped((ShippingInfo) data);
            case "ITEMS_DELIVERED" -> handleItemsDelivered((DeliveryInfo) data);
        }
    }

    private void handleOrderCreated(Order order) {
        // Coordinate multiple services
        inventory.reserve(order.getItems());
        payment.initiateCharge(order);
        analytics.track("order_created", order);
    }

    private void handlePaymentCompleted(PaymentResult result) {
        Order order = result.getOrder();
        shipping.schedule(order);
        notification.sendPaymentConfirmation(order);
        analytics.track("payment_completed", result);
    }

    private void handlePaymentFailed(PaymentResult result) {
        Order order = result.getOrder();
        inventory.release(order.getItems());
        notification.sendPaymentFailure(order, result.getError());
        analytics.track("payment_failed", result);
    }

    private void handleItemsShipped(ShippingInfo info) {
        notification.sendShippingNotification(info);
        analytics.track("items_shipped", info);
    }

    private void handleItemsDelivered(DeliveryInfo info) {
        notification.sendDeliveryConfirmation(info);
        analytics.track("delivery_completed", info);
    }
}
```

### 3. Colleague Classes

```java
@ApplicationScoped
public class PaymentService {

    @Inject
    OrderMediator mediator;

    public void processPayment(Order order) {
        try {
            PaymentResult result = chargePayment(order);
            result.setOrder(order);
            mediator.notify(this, "PAYMENT_COMPLETED", result);
        } catch (PaymentException e) {
            PaymentResult failure = PaymentResult.failed(order, e.getMessage());
            mediator.notify(this, "PAYMENT_FAILED", failure);
        }
    }
}

@ApplicationScoped
public class ShippingService {

    @Inject
    OrderMediator mediator;

    public void confirmDelivery(String trackingNumber) {
        DeliveryInfo info = updateDeliveryStatus(trackingNumber);
        mediator.notify(this, "ITEMS_DELIVERED", info);
    }
}
```

### 4. Alternative: CDI Events as Mediator

CDI Events provide a built-in mediator mechanism:

```java
// Instead of explicit mediator, use CDI Events
@ApplicationScoped
public class PaymentService {

    @Inject
    Event<PaymentCompletedEvent> paymentCompleted;

    @Inject
    Event<PaymentFailedEvent> paymentFailed;

    public void processPayment(Order order) {
        try {
            PaymentResult result = chargePayment(order);
            paymentCompleted.fire(new PaymentCompletedEvent(order, result));
        } catch (PaymentException e) {
            paymentFailed.fire(new PaymentFailedEvent(order, e));
        }
    }
}

// Observers act as coordinated colleagues
@ApplicationScoped
public class ShippingCoordinator {

    public void onPaymentCompleted(@Observes PaymentCompletedEvent event) {
        scheduleShipping(event.getOrder());
    }
}

@ApplicationScoped
public class InventoryCoordinator {

    public void onPaymentFailed(@Observes PaymentFailedEvent event) {
        releaseReservation(event.getOrder());
    }
}
```

### 5. Chat Room Example

```java
public interface ChatMediator {
    void sendMessage(String message, User sender);
    void addUser(User user);
}

@ApplicationScoped
public class ChatRoom implements ChatMediator {

    private final Set<User> users = ConcurrentHashMap.newKeySet();

    @Override
    public void addUser(User user) {
        users.add(user);
        user.setMediator(this);
    }

    @Override
    public void sendMessage(String message, User sender) {
        users.stream()
            .filter(u -> !u.equals(sender))
            .forEach(u -> u.receive(message, sender));
    }
}

public class User {
    private String name;
    private ChatMediator mediator;

    public void send(String message) {
        mediator.sendMessage(message, this);
    }

    public void receive(String message, User from) {
        System.out.println(name + " received from " + from.getName() + ": " + message);
    }

    void setMediator(ChatMediator mediator) {
        this.mediator = mediator;
    }
}
```

## When to Use

✅ **Use Mediator when:**

- Objects communicate in complex but well-defined ways
- Reusing objects is difficult due to many interconnections
- Behavior is distributed among many classes

❌ **Avoid Mediator when:**

- Communication is simple and direct
- Mediator becomes a "god object" with too much logic (consider CDI Events instead)
