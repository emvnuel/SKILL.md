# Facade Pattern - Jakarta EE Cookbook

## Intent

Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.

## Jakarta EE Implementation

### 1. Order Processing Facade

```java
// Facade - simplifies complex subsystem interactions
@ApplicationScoped
public class OrderFacade {

    @Inject
    InventoryService inventory;

    @Inject
    PaymentService payment;

    @Inject
    ShippingService shipping;

    @Inject
    NotificationService notification;

    @Inject
    OrderRepository orders;

    @Transactional
    public OrderResult processOrder(OrderRequest request) {
        // Validate inventory
        inventory.reserve(request.getItems());

        try {
            // Process payment
            PaymentResult paymentResult = payment.charge(request.getPayment());

            // Schedule shipping
            ShippingInfo shippingInfo = shipping.schedule(
                request.getShippingAddress(),
                request.getItems()
            );

            // Create and save order
            Order order = Order.builder()
                .items(request.getItems())
                .paymentId(paymentResult.getTransactionId())
                .shippingId(shippingInfo.getTrackingNumber())
                .build();
            orders.save(order);

            // Send notifications
            notification.sendOrderConfirmation(order);

            return OrderResult.success(order);

        } catch (Exception e) {
            inventory.release(request.getItems());
            throw e;
        }
    }

    public Order cancelOrder(Long orderId) {
        Order order = orders.findById(orderId);
        payment.refund(order.getPaymentId());
        shipping.cancel(order.getShippingId());
        inventory.release(order.getItems());
        order.setStatus(OrderStatus.CANCELLED);
        notification.sendCancellationNotice(order);
        return orders.save(order);
    }
}
```

### 2. Using the Facade in REST Resource

```java
@Path("/orders")
@ApplicationScoped
public class OrderResource {

    @Inject
    OrderFacade orderFacade;  // Simple interface to complex operations

    @POST
    @Transactional
    public Response createOrder(OrderRequest request) {
        OrderResult result = orderFacade.processOrder(request);
        return Response.status(Status.CREATED).entity(result).build();
    }

    @DELETE
    @Path("/{id}")
    public Response cancelOrder(@PathParam("id") Long orderId) {
        Order order = orderFacade.cancelOrder(orderId);
        return Response.ok(order).build();
    }
}
```

### 3. User Registration Facade

```java
@ApplicationScoped
public class RegistrationFacade {

    @Inject UserService userService;
    @Inject EmailService emailService;
    @Inject ProfileService profileService;
    @Inject WelcomePackageService welcomeService;
    @Inject AnalyticsService analytics;

    @Transactional
    public RegistrationResult register(RegistrationRequest request) {
        // One simple method hides complex subsystem interactions
        User user = userService.create(request.getCredentials());
        Profile profile = profileService.createDefault(user);
        emailService.sendVerification(user);
        welcomeService.scheduleWelcomePackage(user);
        analytics.trackRegistration(user);

        return new RegistrationResult(user, profile);
    }
}
```

## When to Use

✅ **Use Facade when:**

- Simplifying access to complex subsystems
- Reducing coupling between clients and subsystem components
- Layering your application (presentation → service facade → domain)
- Providing a simple default view of the subsystem

❌ **Avoid Facade when:**

- Subsystem is already simple
- Clients need fine-grained control over subsystem components
