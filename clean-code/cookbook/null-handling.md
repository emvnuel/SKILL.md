# Null Handling - Clean Code Cookbook

## Intent

Don't return null. Don't pass null. Null is a billion-dollar mistake. Use Optional, empty collections, or the Null Object pattern instead.

---

## Don't Return Null

### Before - Returning Null

```java
public List<Customer> findCustomers(String region) {
    List<Customer> customers = repository.findByRegion(region);
    if (customers.isEmpty()) {
        return null;  // BAD - forces null checks
    }
    return customers;
}

// Every caller must check for null
List<Customer> customers = findCustomers("EAST");
if (customers != null) {  // Defensive check
    for (Customer c : customers) {
        // ...
    }
}
```

### After - Return Empty Collection

```java
public List<Customer> findCustomers(String region) {
    return repository.findByRegion(region);  // Returns empty list, never null
}

// Caller doesn't need null check
List<Customer> customers = findCustomers("EAST");
for (Customer c : customers) {  // Works even if empty
    // ...
}
```

---

## Use Optional for Single Values

### Before - Null Return

```java
public Customer findById(Long id) {
    return repository.findById(id);  // Might return null
}

// Caller
Customer customer = findById(123L);
if (customer != null) {  // Required but easy to forget
    process(customer);
}
```

### After - Optional

```java
public Optional<Customer> findById(Long id) {
    return Optional.ofNullable(repository.findById(id));
}

// Caller - must handle absence explicitly
findById(123L)
    .ifPresentOrElse(
        this::process,
        () -> log.warn("Customer not found")
    );

// Or with default
Customer customer = findById(123L)
    .orElseThrow(() -> new CustomerNotFoundException(123L));

// Or with fallback
Customer customer = findById(123L)
    .orElse(Customer.anonymous());
```

---

## Null Object Pattern

Use when you need an object that does nothing safely.

### Before - Null Checks

```java
public interface DiscountStrategy {
    Money calculateDiscount(Order order);
}

public class OrderService {
    public Money getDiscount(Order order) {
        DiscountStrategy strategy = findStrategy(order.getCustomerType());
        if (strategy != null) {
            return strategy.calculateDiscount(order);
        }
        return Money.ZERO;  // Null check scattered everywhere
    }
}
```

### After - Null Object

```java
public interface DiscountStrategy {
    Money calculateDiscount(Order order);

    // Null Object as constant
    DiscountStrategy NONE = order -> Money.ZERO;
}

public class OrderService {
    public Money getDiscount(Order order) {
        DiscountStrategy strategy = findStrategy(order.getCustomerType());
        return strategy.calculateDiscount(order);  // Never null
    }

    private DiscountStrategy findStrategy(CustomerType type) {
        return strategies.stream()
            .filter(s -> s.supports(type))
            .findFirst()
            .orElse(DiscountStrategy.NONE);  // Return null object
    }
}
```

---

## Don't Pass Null

### Before - Null Arguments

```java
public Money calculateShipping(Address from, Address to) {
    // What if from or to is null?
    return shippingService.calculate(from, to);
}

// Caller can pass null
Money shipping = calculateShipping(null, customerAddress);
```

### After - Validate or Overload

```java
// Option 1: Validate early
public Money calculateShipping(Address from, Address to) {
    requireNonNull(from, "From address cannot be null");
    requireNonNull(to, "To address cannot be null");
    return shippingService.calculate(from, to);
}

// Option 2: Overload for optional parameters
public Money calculateShipping(Address to) {
    return calculateShipping(defaultWarehouse(), to);
}

public Money calculateShipping(Address from, Address to) {
    return shippingService.calculate(from, to);
}

// Option 3: Use builder or parameter object
public Money calculateShipping(ShippingRequest request) {
    return shippingService.calculate(request);
}

ShippingRequest request = ShippingRequest.builder()
    .to(customerAddress)
    .from(warehouse)  // Optional
    .build();
```

---

## Defensive Programming vs Clean Design

### Defensive Style (cluttered)

```java
public void processOrder(Order order) {
    if (order == null) return;
    if (order.getItems() == null) return;
    if (order.getCustomer() == null) return;

    for (Item item : order.getItems()) {
        if (item == null) continue;
        if (item.getProduct() == null) continue;
        // finally do something
    }
}
```

### Clean Design (null-safe by construction)

```java
// Order enforces invariants
public class Order {
    private final Customer customer;
    private final List<Item> items;

    public Order(Customer customer, List<Item> items) {
        this.customer = requireNonNull(customer, "Customer required");
        this.items = items != null ? List.copyOf(items) : List.of();
    }

    public Customer getCustomer() { return customer; }
    public List<Item> getItems() { return items; }  // Never null
}

// Service can trust its inputs
public void processOrder(Order order) {
    requireNonNull(order, "Order required");

    for (Item item : order.getItems()) {  // Safe iteration
        processItem(item);
    }
}
```

---

## Jakarta EE Patterns

### Bean Validation

```java
public void createCustomer(
    @NotNull @Valid CustomerRequest request
) {
    // Bean validation ensures not null
    Customer customer = mapper.toEntity(request);
    repository.save(customer);
}
```

### Optional with JPA

```java
@Repository
public class CustomerRepository {

    @PersistenceContext
    EntityManager em;

    public Optional<Customer> findById(Long id) {
        return Optional.ofNullable(
            em.find(Customer.class, id)
        );
    }

    public Optional<Customer> findByEmail(String email) {
        try {
            return Optional.of(
                em.createQuery("SELECT c FROM Customer c WHERE c.email = :email", Customer.class)
                    .setParameter("email", email)
                    .getSingleResult()
            );
        } catch (NoResultException e) {
            return Optional.empty();
        }
    }
}
```

### CDI Optional Injection

```java
@ApplicationScoped
public class NotificationService {

    @Inject
    Instance<EmailService> emailService;  // May not be available

    public void notify(Customer customer, String message) {
        if (emailService.isResolvable()) {
            emailService.get().send(customer.getEmail(), message);
        }
        // Fallback or skip silently
    }
}
```

---

## Optional Best Practices

| Use Case                     | Approach                                   |
| ---------------------------- | ------------------------------------------ |
| Return single optional value | `Optional<T>`                              |
| Return possibly empty list   | Empty `List<T>`, never null                |
| Method parameter             | Overload or parameter object, NOT Optional |
| Class field                  | Initialize to non-null, NOT Optional       |
| Primitive types              | `OptionalInt`, `OptionalLong`, etc.        |

### Anti-patterns

```java
// Bad - Optional as field
private Optional<Address> address;

// Good - nullable field with clear getter
private Address address;
public Optional<Address> getAddress() {
    return Optional.ofNullable(address);
}

// Bad - Optional as method parameter
public void save(Optional<String> name);

// Good - overload or null
public void save(String name);  // null for no name
public void save();  // no name parameter
```

---

## Related Recipes

- [Error Handling](./error-handling.md): Exceptions over return codes
- [Small Functions](./small-functions.md): Guard clauses for validation
