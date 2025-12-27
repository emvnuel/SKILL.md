# CDI Bean Design with CDD

## Principle

CDI services should stay within cognitive load limits (≤7 points for services) and delegate to domain objects.

## Service Layer Pattern

```java
@ApplicationScoped
public class OrderService {

    @Inject
    private OrderRepository orders;  // +1

    @Inject
    private CustomerRepository customers;  // +1

    @Inject
    private Event<OrderCreatedEvent> orderCreated;  // +1

    @Transactional
    public Order create(CreateOrderRequest request) {
        Customer customer = customers.findById(request.customerId())  // +1
            .orElseThrow(CustomerNotFoundException::new);

        Order order = request.toEntity(customer);  // +1
        Order saved = orders.save(order);

        orderCreated.fire(new OrderCreatedEvent(saved));  // +1

        return saved;
    }
    // Total: 6 points ✓
}
```

## When to Split Services

If a service exceeds limits, extract focused components:

```java
// ❌ Too many concerns
@ApplicationScoped
public class OrderService {
    // Payment processing
    // Inventory management
    // Notification sending
    // Audit logging
    // 12+ points
}

// ✓ Focused services
@ApplicationScoped
public class OrderService {
    @Inject private OrderFulfillmentCoordinator coordinator;

    @Transactional
    public FulfillmentResult fulfill(Long orderId) {
        Order order = findOrThrow(orderId);
        return coordinator.fulfill(order);
    }
}

@ApplicationScoped
class OrderFulfillmentCoordinator {
    @Inject private InventoryService inventory;
    @Inject private PaymentService payments;
    @Inject private NotificationService notifications;

    public FulfillmentResult fulfill(Order order) {
        inventory.reserve(order);
        payments.capture(order);
        notifications.sendConfirmation(order);
        return FulfillmentResult.success(order);
    }
}
```

## CDI Scope Selection

| Scope                | Use Case                            |
| -------------------- | ----------------------------------- |
| `@ApplicationScoped` | Stateless services, singletons      |
| `@RequestScoped`     | Per-request state, JAX-RS resources |
| `@Dependent`         | New instance per injection point    |
| `@SessionScoped`     | User session state                  |

```java
@ApplicationScoped  // Stateless, shared
public class ProductService {
    @Inject private ProductRepository products;

    public Product findById(Long id) {
        return products.findById(id).orElseThrow();
    }
}

@RequestScoped  // Per-request, for resources
public class ProductResource {
    @Inject private ProductService service;
}
```

## Event-Driven Decoupling

```java
// Fire events instead of direct calls
@ApplicationScoped
public class OrderService {

    @Inject
    private Event<OrderCreatedEvent> orderCreated;

    @Transactional
    public Order create(CreateOrderRequest request) {
        Order order = /* ... */;
        orders.save(order);

        // Decouple from side effects
        orderCreated.fire(new OrderCreatedEvent(order));

        return order;
    }
}

// Separate observer handles side effects
@ApplicationScoped
public class OrderNotificationHandler {

    @Inject
    private EmailService emailService;

    public void onOrderCreated(
            @Observes @Priority(100) OrderCreatedEvent event) {
        emailService.sendOrderConfirmation(event.getOrder());
    }
}
```

## Qualifier for Multiple Implementations

```java
// Qualifier annotation
@Qualifier
@Retention(RUNTIME)
@Target({FIELD, TYPE, METHOD, PARAMETER})
public @interface PaymentType {
    String value();
}

// Implementations
@ApplicationScoped
@PaymentType("CREDIT_CARD")
public class CreditCardPaymentProcessor implements PaymentProcessor { }

@ApplicationScoped
@PaymentType("PIX")
public class PixPaymentProcessor implements PaymentProcessor { }

// Injection
@Inject
@PaymentType("CREDIT_CARD")
private PaymentProcessor creditCardProcessor;
```

## Cognitive Load Analysis Template

```java
@ApplicationScoped
public class ExampleService {

    @Inject private DependencyA depA;    // +1
    @Inject private DependencyB depB;    // +1

    public void doSomething() {
        if (condition) {                  // +1
            // ...
        }

        for (Item item : items) {         // +1
            try {                         // +1
                // ...
            } catch (Exception e) {       // +1
                // ...
            }
        }
    }
    // Total: 6 points ✓
}
```

## Related Entries

- [cognitive-load-limits](cognitive-load-limits.md) - Counting points
- [extract-abstractions](extract-abstractions.md) - When to split
