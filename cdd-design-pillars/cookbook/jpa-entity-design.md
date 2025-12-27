# JPA Entity Design with CDD

## Principle

Entities can have higher cognitive load (≤9 points) and should encapsulate business logic operating on their state.

## Rich Entity Pattern

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    private Instant createdAt;
    private Instant expiresAt;

    // JPA requires
    protected Order() {}

    // Business constructor - valid state only
    public Order(Customer customer, List<OrderItem> items) {
        Objects.requireNonNull(customer, "Customer required");
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("Order must have items");
        }

        this.customer = customer;
        this.status = OrderStatus.PENDING;
        this.createdAt = Instant.now();
        this.expiresAt = createdAt.plus(Duration.ofHours(24));

        items.forEach(this::addItem);
    }

    // ✓ Encapsulated state modification
    public void addItem(OrderItem item) {
        item.setOrder(this);
        this.items.add(item);
    }

    // ✓ Logic on owned data
    public boolean isExpired() {
        return Instant.now().isAfter(expiresAt);
    }

    public boolean canBeModified() {
        return status == OrderStatus.PENDING && !isExpired();
    }

    // ✓ State transition with validation
    public void confirm() {
        if (!canBeModified()) {
            throw new OrderCannotBeConfirmedException(this);
        }
        this.status = OrderStatus.CONFIRMED;
    }

    public void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new CannotCancelShippedOrderException(this);
        }
        this.status = OrderStatus.CANCELLED;
    }

    // ✓ Calculation in entity
    public BigDecimal getTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    // Read-only access
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }

    public Long getId() { return id; }
    public Customer getCustomer() { return customer; }
    public OrderStatus getStatus() { return status; }
}
```

## Value Object as Embeddable

```java
@Embeddable
public class Money {

    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    private Currency currency;

    protected Money() {}

    public Money(BigDecimal amount, Currency currency) {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        if (other.currency != this.currency) {
            throw new CurrencyMismatchException(this.currency, other.currency);
        }
        return new Money(amount.add(other.amount), currency);
    }

    public Money multiply(int quantity) {
        return new Money(amount.multiply(BigDecimal.valueOf(quantity)), currency);
    }

    public boolean isGreaterThan(Money other) {
        return this.amount.compareTo(other.amount) > 0;
    }

    public String formatted() {
        return NumberFormat.getCurrencyInstance()
            .format(amount);
    }
}

// Usage in entity
@Entity
public class Product {

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount", column = @Column(name = "price_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "price_currency"))
    })
    private Money price;
}
```

## Entity Lifecycle Hooks

```java
@Entity
public class AuditedEntity {

    @Column(updatable = false)
    private Instant createdAt;

    private Instant updatedAt;

    @PrePersist
    void onCreate() {
        this.createdAt = Instant.now();
        this.updatedAt = this.createdAt;
    }

    @PreUpdate
    void onUpdate() {
        this.updatedAt = Instant.now();
    }
}
```

## Cognitive Load Analysis

```java
@Entity
public class Subscription {
    // Dependencies: 0 (entities don't inject)

    public boolean isActive() {
        return status == Status.ACTIVE          // +1 (comparison)
            && !LocalDate.now().isAfter(endDate); // Operations on standard types don't count
    }

    public void renew() {
        if (!canRenew()) {                      // +1
            throw new CannotRenewException(this);
        }
        this.endDate = endDate.plusMonths(plan.getBillingCycle());
    }

    public void cancel() {
        if (status == Status.CANCELLED) {       // +1
            throw new AlreadyCancelledException(this);
        }
        this.status = Status.CANCELLED;
    }

    // More methods...
}
// Entities can reach 9 points comfortably
```

## Related Entries

- [valid-state-only](valid-state-only.md) - Constructor patterns
- [concentrate-operations](concentrate-operations.md) - Logic in entities
