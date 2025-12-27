# State Pattern - Jakarta EE Cookbook

## Intent

Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

## Jakarta EE Implementation

### 1. State Interface

```java
public interface OrderState {
    void pay(Order order);
    void ship(Order order);
    void deliver(Order order);
    void cancel(Order order);
    String getStatus();
}
```

### 2. Concrete States

```java
public class PendingState implements OrderState {

    @Override
    public void pay(Order order) {
        order.setState(new PaidState());
    }

    @Override
    public void ship(Order order) {
        throw new IllegalStateException("Cannot ship unpaid order");
    }

    @Override
    public void deliver(Order order) {
        throw new IllegalStateException("Cannot deliver unpaid order");
    }

    @Override
    public void cancel(Order order) {
        order.setState(new CancelledState());
    }

    @Override
    public String getStatus() {
        return "PENDING";
    }
}

public class PaidState implements OrderState {

    @Override
    public void pay(Order order) {
        throw new IllegalStateException("Order already paid");
    }

    @Override
    public void ship(Order order) {
        order.setState(new ShippedState());
        order.notifyShipping();
    }

    @Override
    public void deliver(Order order) {
        throw new IllegalStateException("Order not shipped yet");
    }

    @Override
    public void cancel(Order order) {
        order.refund();
        order.setState(new CancelledState());
    }

    @Override
    public String getStatus() {
        return "PAID";
    }
}

public class ShippedState implements OrderState {

    @Override
    public void pay(Order order) {
        throw new IllegalStateException("Order already paid");
    }

    @Override
    public void ship(Order order) {
        throw new IllegalStateException("Order already shipped");
    }

    @Override
    public void deliver(Order order) {
        order.setState(new DeliveredState());
        order.notifyDelivery();
    }

    @Override
    public void cancel(Order order) {
        throw new IllegalStateException("Cannot cancel shipped order");
    }

    @Override
    public String getStatus() {
        return "SHIPPED";
    }
}

public class DeliveredState implements OrderState {

    @Override
    public void pay(Order order) {
        throw new IllegalStateException("Order completed");
    }

    @Override
    public void ship(Order order) {
        throw new IllegalStateException("Order completed");
    }

    @Override
    public void deliver(Order order) {
        throw new IllegalStateException("Order already delivered");
    }

    @Override
    public void cancel(Order order) {
        throw new IllegalStateException("Cannot cancel delivered order");
    }

    @Override
    public String getStatus() {
        return "DELIVERED";
    }
}
```

### 3. Context Class

```java
@Entity
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    @Transient
    private OrderState state = new PendingState();

    @Column(name = "status")
    private String statusName = "PENDING";

    // Delegate to state
    public void pay() {
        state.pay(this);
        this.statusName = state.getStatus();
    }

    public void ship() {
        state.ship(this);
        this.statusName = state.getStatus();
    }

    public void deliver() {
        state.deliver(this);
        this.statusName = state.getStatus();
    }

    public void cancel() {
        state.cancel(this);
        this.statusName = state.getStatus();
    }

    public String getStatus() {
        return state.getStatus();
    }

    // Package-private for state classes
    void setState(OrderState state) {
        this.state = state;
    }

    void notifyShipping() {
        // Send shipping notification
    }

    void notifyDelivery() {
        // Send delivery notification
    }

    void refund() {
        // Process refund
    }

    // Restore state from database
    @PostLoad
    void restoreState() {
        this.state = switch (statusName) {
            case "PENDING" -> new PendingState();
            case "PAID" -> new PaidState();
            case "SHIPPED" -> new ShippedState();
            case "DELIVERED" -> new DeliveredState();
            case "CANCELLED" -> new CancelledState();
            default -> throw new IllegalStateException("Unknown status: " + statusName);
        };
    }
}
```

### 4. Usage

```java
@ApplicationScoped
public class OrderService {

    @Inject
    OrderRepository orders;

    @Transactional
    public Order processPayment(Long orderId, PaymentDetails payment) {
        Order order = orders.findById(orderId);
        // Payment processing...
        order.pay();  // State handles transition
        return orders.save(order);
    }

    @Transactional
    public Order shipOrder(Long orderId) {
        Order order = orders.findById(orderId);
        order.ship();  // Throws if not in PAID state
        return orders.save(order);
    }
}
```

## When to Use

✅ **Use State when:**

- Object behavior depends on its state
- You have complex conditional logic based on state
- State transitions follow specific rules

❌ **Avoid State when:**

- Few states with simple transitions
- State changes are rare
