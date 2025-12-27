# Concentrate Operations in the Owning Class

## Principle

> Operations on data whose type is a language primitive/standard should live inside the class that owns the data.

This is the essence of encapsulation from Barbara Liskov's Abstract Data Types paper.

## Problem

Logic operating on a class's internal state lives outside the class.

```java
// ❌ BAD: Logic on Order's data lives outside Order
@ApplicationScoped
public class OrderExpirationChecker {

    public boolean isExpired(Order order) {
        // Operating on Order's state externally
        return order.getExpiresAt().isBefore(Instant.now());
    }

    public boolean acceptsModification(Order order) {
        // More external logic on Order's state
        return !isExpired(order) && order.getStatus() == OrderStatus.PENDING;
    }
}

// Usage scattered everywhere
if (expirationChecker.isExpired(order)) {
    throw new OrderExpiredException();
}
```

## Solution

Move operations to the class that owns the data.

```java
@Entity
public class Order {

    private Instant expiresAt;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @ManyToOne
    private Customer customer;

    // ✓ Logic on Instant (standard type) lives in Order
    public boolean isExpired() {
        return expiresAt.isBefore(Instant.now());
    }

    // ✓ Parameterized version for testing
    public boolean isExpiredAt(Instant referenceTime) {
        return expiresAt.isBefore(referenceTime);
    }

    // ✓ Combined rule in owning class
    public boolean acceptsModification() {
        return !isExpired() && status == OrderStatus.PENDING;
    }

    public void modify(OrderModification modification) {
        if (!acceptsModification()) {
            throw new OrderCannotBeModifiedException(this);
        }
        // apply modification
    }
}

// Usage is now simple
order.modify(modification);  // Throws if not allowed
```

## The Chain Rule

Apply the same rule recursively. If a type operates on language primitives, delegate there.

```java
// ❌ BAD: Accessing member's internal collection
public Set<PaymentMethod> filterDesiredPayments(User user) {
    return this.paymentMethods.stream()
        .filter(pm -> user.getDesiredPaymentMethods().contains(pm))  // Accessing user's state
        .collect(Collectors.toSet());
}

// ✓ GOOD: Ask the User
public Set<PaymentMethod> filterDesiredPayments(User user) {
    return this.paymentMethods.stream()
        .filter(user::acceptsPaymentMethod)  // Delegate to User
        .collect(Collectors.toSet());
}
```

```java
@Entity
public class User {

    @ElementCollection
    private Set<PaymentMethod> desiredPaymentMethods;

    // ✓ Operation on contained Set lives in User
    public boolean acceptsPaymentMethod(PaymentMethod method) {
        return desiredPaymentMethods.contains(method);
    }
}
```

## Jakarta EE Example

### Entity with Encapsulated Operations

```java
@Entity
public class Subscription {

    @ManyToOne
    private Plan plan;

    private LocalDate startDate;
    private LocalDate endDate;

    @Enumerated(EnumType.STRING)
    private SubscriptionStatus status;

    // ✓ Operations on LocalDate live here
    public boolean isActive() {
        return status == SubscriptionStatus.ACTIVE
            && !LocalDate.now().isAfter(endDate);
    }

    public boolean isInTrialPeriod() {
        return status == SubscriptionStatus.TRIAL
            && LocalDate.now().isBefore(startDate.plusDays(14));
    }

    // ✓ Operations on Plan delegate to Plan
    public boolean allows(Feature feature) {
        return plan.includes(feature);
    }

    public int getRemainingDays() {
        return (int) ChronoUnit.DAYS.between(LocalDate.now(), endDate);
    }

    public void renew() {
        if (!isActive()) {
            throw new CannotRenewInactiveSubscriptionException(this);
        }
        this.endDate = endDate.plusMonths(plan.getBillingCycleMonths());
    }
}
```

### Service Uses Domain Logic

```java
@ApplicationScoped
public class FeatureAccessService {

    @Inject
    private SubscriptionRepository subscriptions;

    public void checkAccess(User user, Feature feature) {
        Subscription sub = subscriptions.findActiveByUser(user)
            .orElseThrow(NoActiveSubscriptionException::new);

        // ✓ Ask the domain object
        if (!sub.allows(feature)) {
            throw new FeatureNotIncludedException(feature, sub);
        }
    }
}
// Total: 3 points ✓ - Simple because logic is in domain
```

## Benefits

- **Distributed complexity**: Logic spread across domain objects
- **Reduced service complexity**: Services orchestrate, don't calculate
- **Reusable rules**: Same logic available everywhere
- **Testable**: Domain logic tested in isolation

## Related Entries

- [favor-encapsulation](favor-encapsulation.md) - Keep state private
- [cognitive-load-limits](cognitive-load-limits.md) - Measure the result
