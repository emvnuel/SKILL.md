# Create Only Valid References

## Principle

> Only create references with valid state. Every language offers mechanisms for this — constructors in OOP languages.

## Problem

Objects created in invalid states lead to null checks scattered throughout the codebase and runtime errors.

```java
// ❌ BAD: Object can exist in invalid state
public class Order {
    private Customer customer;
    private List<OrderItem> items;
    private OrderStatus status;

    public Order() {}  // Empty object is invalid

    public void setCustomer(Customer c) { this.customer = c; }
    public void setItems(List<OrderItem> items) { this.items = items; }
}

// Code everywhere needs to check validity
if (order.getCustomer() != null && order.getItems() != null) {
    // proceed
}
```

## Solution

Use constructors and factory methods to enforce invariants from object creation.

```java
@Entity
public class Order {

    @Id @GeneratedValue
    private Long id;

    @ManyToOne(optional = false)
    private Customer customer;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    // JPA requires no-arg constructor
    protected Order() {}

    // ✓ Business constructor enforces valid state
    public Order(Customer customer, List<OrderItem> items) {
        Objects.requireNonNull(customer, "Customer is required");
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }

        this.customer = customer;
        this.items = new ArrayList<>(items);
        this.status = OrderStatus.PENDING;
    }

    // No setters for required fields - immutable after creation
}
```

### Factory Methods for Complex Construction

```java
public class Subscription {

    private final Plan plan;
    private final Customer customer;
    private final LocalDate startDate;
    private final LocalDate endDate;

    private Subscription(Plan plan, Customer customer,
                         LocalDate startDate, LocalDate endDate) {
        this.plan = plan;
        this.customer = customer;
        this.startDate = startDate;
        this.endDate = endDate;
    }

    // ✓ Factory methods express intent and enforce rules
    public static Subscription monthly(Customer customer, Plan plan) {
        LocalDate start = LocalDate.now();
        return new Subscription(plan, customer, start, start.plusMonths(1));
    }

    public static Subscription annual(Customer customer, Plan plan) {
        LocalDate start = LocalDate.now();
        return new Subscription(plan, customer, start, start.plusYears(1));
    }

    public static Subscription trial(Customer customer) {
        LocalDate start = LocalDate.now();
        return new Subscription(Plan.TRIAL, customer, start, start.plusDays(14));
    }
}

// Usage - intent is clear
Subscription sub = Subscription.trial(customer);
```

### With Bean Validation

```java
public record CreateAccountRequest(
    @NotBlank @Email
    String email,

    @NotBlank @Size(min = 8, max = 100)
    String password,

    @NotBlank
    String fullName
) {
    // Compact constructor validates on creation
    public CreateAccountRequest {
        email = email.toLowerCase().trim();
        fullName = fullName.trim();
    }

    public Account toEntity(PasswordEncoder encoder) {
        return new Account(email, encoder.encode(password), fullName);
    }
}
```

## Protected Constructor for JPA

```java
@Entity
public class Product {

    @Id @GeneratedValue
    private Long id;

    private String name;
    private BigDecimal price;

    // ✓ Protected: JPA can access, clients cannot
    protected Product() {}

    public Product(String name, BigDecimal price) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name required");
        }
        if (price == null || price.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Price must be positive");
        }
        this.name = name;
        this.price = price;
    }
}
```

## Benefits

- **No null checks**: Objects are always valid
- **Fail fast**: Errors caught at creation, not later
- **Self-documenting**: Constructor shows what's required
- **Immutability**: No setters for invariants

## Related Entries

- [explicit-business-rules](explicit-business-rules.md) - Express rules in constructors
- [validation-patterns](validation-patterns.md) - Bean Validation integration
