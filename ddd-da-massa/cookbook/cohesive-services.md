# 100% Cohesive Services

## Principle

> Every method in a service must use ALL injected dependencies, just like controllers.

## Problem

Services with partially used dependencies:

```java
// ❌ Not cohesive
@ApplicationScoped
public class OrderService {

    @Inject private OrderRepository orders;      // Used by 4 methods
    @Inject private PaymentService payments;     // Used by 1 method
    @Inject private ShippingService shipping;    // Used by 1 method
    @Inject private NotificationService notify;  // Used by 2 methods
    @Inject private ReportService reports;       // Used by 1 method

    public Order create(...) { /* orders, notify */ }
    public void confirm(...) { /* orders, payments, notify */ }
    public void ship(...) { /* orders, shipping */ }
    public Report generate(...) { /* orders, reports */ }
}
```

## Solution

Split into focused services:

```java
@ApplicationScoped
public class OrderCreationService {

    @Inject private OrderRepository orders;
    @Inject private NotificationService notify;

    @Transactional
    public Order create(CreateOrderRequest request, Customer customer) {
        Order order = request.toEntity(customer);
        orders.save(order);
        notify.sendOrderConfirmation(order);
        return order;
    }
}
// Both deps used ✓

@ApplicationScoped
public class OrderPaymentService {

    @Inject private OrderRepository orders;
    @Inject private PaymentService payments;
    @Inject private NotificationService notify;

    @Transactional
    public void confirm(Long orderId, PaymentRequest request) {
        Order order = orders.findById(orderId)
            .orElseThrow(NotFoundException::new);

        payments.process(order, request);
        order.confirm();
        orders.save(order);

        notify.sendPaymentConfirmation(order);
    }
}
// All 3 deps used ✓

@ApplicationScoped
public class OrderShippingService {

    @Inject private OrderRepository orders;
    @Inject private ShippingService shipping;

    @Transactional
    public void ship(Long orderId) {
        Order order = orders.findById(orderId)
            .orElseThrow(NotFoundException::new);

        shipping.createShipment(order);
        order.ship();
        orders.save(order);
    }
}
// Both deps used ✓
```

## Package Organization

```
services/
├── orders/
│   ├── OrderCreationService.java
│   ├── OrderPaymentService.java
│   └── OrderShippingService.java
└── reports/
    └── OrderReportService.java
```

## Cognitive Load Benefit

Focused services stay within 7 points:

```java
@ApplicationScoped
public class OrderPaymentService {
    @Inject private OrderRepository orders;      // +1
    @Inject private PaymentService payments;     // +1
    @Inject private NotificationService notify;  // +1

    public void confirm(Long orderId, PaymentRequest request) {
        Order order = orders.findById(orderId)
            .orElseThrow(NotFoundException::new);  // +1

        payments.process(order, request);          // uses injected
        order.confirm();
        orders.save(order);
        notify.sendPaymentConfirmation(order);
    }
}
// Total: 4 points ✓
```

## When Single Method is OK

If service has ONE method, that's fine — it's still 100% cohesive:

```java
@ApplicationScoped
public class OrderCancellationService {

    @Inject private OrderRepository orders;
    @Inject private RefundService refunds;
    @Inject private NotificationService notify;

    @Transactional
    public void cancel(Long orderId, CancellationReason reason) {
        Order order = orders.findById(orderId)
            .orElseThrow(NotFoundException::new);

        order.cancel(reason);
        refunds.process(order);
        orders.save(order);
        notify.sendCancellationNotice(order);
    }
}
// Single method, all 3 deps used ✓
```

## Related Entries

- [cohesive-resources](cohesive-resources.md) - Same for controllers
- [splitting-load](splitting-load.md) - When to split further
