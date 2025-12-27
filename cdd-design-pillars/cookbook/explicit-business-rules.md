# Explicit Business Rules

## Principle

> Business rules must be declared explicitly in our application. Signal clearly when any problem occurs during flow execution.

## Problem

Business rules hidden in conditionals, scattered across layers, or implicit in data make code hard to understand and maintain.

```java
// ❌ BAD: Business rule is implicit
public void processPayment(Order order) {
    if (order.getTotal().compareTo(new BigDecimal("10000")) > 0
        && !order.getCustomer().isPremium()
        && order.getPaymentMethod().equals("CREDIT_CARD")) {
        // What rule is this? Why these conditions?
        throw new RuntimeException("Cannot process");
    }
}
```

## Solution

Make business rules explicit through named methods, custom exceptions, and domain events.

### Named Business Rules

```java
@Entity
public class Order {

    private BigDecimal total;

    @ManyToOne
    private Customer customer;

    private static final BigDecimal LARGE_ORDER_THRESHOLD = new BigDecimal("10000");

    // ✓ Rule is named and documented
    public boolean requiresManualApproval() {
        return isLargeOrder() && !customer.isPremium();
    }

    public boolean isLargeOrder() {
        return total.compareTo(LARGE_ORDER_THRESHOLD) > 0;
    }

    // ✓ Validation with explicit error
    public void validateForPayment() {
        if (requiresManualApproval()) {
            throw new OrderRequiresApprovalException(this);
        }
    }
}
```

### Custom Exceptions for Business Violations

```java
// Base for business rule violations
public abstract class BusinessRuleViolationException extends RuntimeException {

    private final String ruleCode;

    protected BusinessRuleViolationException(String ruleCode, String message) {
        super(message);
        this.ruleCode = ruleCode;
    }

    public String getRuleCode() {
        return ruleCode;
    }
}

// Specific violation
public class InsufficientBalanceException extends BusinessRuleViolationException {

    private final BigDecimal required;
    private final BigDecimal available;

    public InsufficientBalanceException(BigDecimal required, BigDecimal available) {
        super("INSUFFICIENT_BALANCE",
            String.format("Required: %s, Available: %s", required, available));
        this.required = required;
        this.available = available;
    }
}
```

### Exception Mapper for JAX-RS

```java
@Provider
public class BusinessRuleExceptionMapper
    implements ExceptionMapper<BusinessRuleViolationException> {

    @Override
    public Response toResponse(BusinessRuleViolationException e) {
        return Response.status(Response.Status.UNPROCESSABLE_ENTITY)
            .entity(new ErrorResponse(e.getRuleCode(), e.getMessage()))
            .build();
    }
}
```

### Domain Service with Explicit Flow

```java
@ApplicationScoped
public class OrderFulfillmentService {

    @Inject
    private InventoryService inventory;

    @Inject
    private PaymentService payments;

    @Inject
    private Event<OrderFulfilledEvent> orderFulfilled;

    @Transactional
    public FulfillmentResult fulfill(Order order) {
        // ✓ Each rule is explicit
        order.validateForFulfillment();

        // ✓ Clear what can fail
        inventory.reserveFor(order)
            .orElseThrow(() -> new InsufficientInventoryException(order));

        // ✓ Named operation
        PaymentResult payment = payments.capture(order);
        if (!payment.isSuccessful()) {
            throw new PaymentFailedException(order, payment);
        }

        // ✓ Explicit state transition
        order.markAsFulfilled();

        // ✓ Domain event for side effects
        orderFulfilled.fire(new OrderFulfilledEvent(order));

        return FulfillmentResult.success(order);
    }
}
```

### Specification Pattern for Complex Rules

```java
public interface OrderSpecification {
    boolean isSatisfiedBy(Order order);
    String getViolationMessage();
}

public class CanBeCancelledSpecification implements OrderSpecification {

    @Override
    public boolean isSatisfiedBy(Order order) {
        return order.getStatus() == OrderStatus.PENDING
            && order.getCreatedAt().isAfter(Instant.now().minus(Duration.ofHours(24)));
    }

    @Override
    public String getViolationMessage() {
        return "Order can only be cancelled within 24 hours of creation while pending";
    }
}

// Usage
public void cancel(Order order) {
    OrderSpecification canCancel = new CanBeCancelledSpecification();
    if (!canCancel.isSatisfiedBy(order)) {
        throw new BusinessRuleViolationException(
            "CANNOT_CANCEL", canCancel.getViolationMessage());
    }
    order.cancel();
}
```

## Benefits

- **Readable**: Rules are named and documented
- **Testable**: Each rule can be unit tested
- **Maintainable**: Changes localized to rule definition
- **Traceable**: Errors include rule codes for debugging

## Related Entries

- [valid-state-only](valid-state-only.md) - Rules in constructors
- [jpa-entity-design](jpa-entity-design.md) - Rules in entities
