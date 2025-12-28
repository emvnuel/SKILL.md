# Single Responsibility Principle - Clean Code Cookbook

## Intent

A class should have one, and only one, reason to change. Each class should have a single responsibility, and that responsibility should be entirely encapsulated by the class.

---

## The Principle

> Gather together the things that change for the same reasons. Separate those things that change for different reasons.

A "reason to change" is an actor or stakeholder who might request changes.

---

## Identifying Multiple Responsibilities

### Signs of SRP Violation

- Class name includes "And" or "Or"
- Difficult to name the class concisely
- Class has methods that don't use each other
- Changes in unrelated features require modifying the class
- Class has many imports from different domains

---

## Before - Multiple Responsibilities

```java
public class Employee {
    private String name;
    private Money salary;

    // Responsibility 1: Business rules (HR)
    public Money calculatePay() {
        return salary;
    }

    // Responsibility 2: Persistence (IT/DBA)
    public void save() {
        Connection conn = DriverManager.getConnection(URL);
        PreparedStatement stmt = conn.prepareStatement(
            "INSERT INTO employees (name, salary) VALUES (?, ?)"
        );
        stmt.setString(1, name);
        stmt.setBigDecimal(2, salary.getAmount());
        stmt.executeUpdate();
    }

    // Responsibility 3: Reporting (Finance)
    public String generatePayStub() {
        return String.format(
            "Employee: %s\nSalary: %s\nTaxes: %s\nNet: %s",
            name, salary, calculateTaxes(), calculateNetPay()
        );
    }
}
```

---

## After - Separated Responsibilities

```java
// Responsibility 1: Employee data and business rules
public class Employee {
    private final EmployeeId id;
    private final String name;
    private final Money salary;
    private final EmployeeType type;

    public Money calculatePay() {
        return type.calculatePay(salary);
    }

    public Money calculateTaxes() {
        return TaxCalculator.calculate(salary, type);
    }
}

// Responsibility 2: Persistence
@ApplicationScoped
public class EmployeeRepository {

    @PersistenceContext
    EntityManager em;

    public void save(Employee employee) {
        em.persist(employee);
    }

    public Optional<Employee> findById(EmployeeId id) {
        return Optional.ofNullable(em.find(Employee.class, id));
    }
}

// Responsibility 3: Report generation
@ApplicationScoped
public class PayStubGenerator {

    public PayStub generate(Employee employee) {
        return new PayStub(
            employee.getName(),
            employee.calculatePay(),
            employee.calculateTaxes(),
            employee.calculateNetPay()
        );
    }

    public String format(PayStub payStub) {
        return String.format(
            "Employee: %s\nSalary: %s\nTaxes: %s\nNet: %s",
            payStub.getName(),
            payStub.getGrossPay(),
            payStub.getTaxes(),
            payStub.getNetPay()
        );
    }
}
```

---

## Layered Architecture Example

Each layer has a single responsibility:

```
┌─────────────────────────────────────┐
│  Presentation (REST Resources)      │  → HTTP handling, validation
├─────────────────────────────────────┤
│  Application (Use Cases/Services)   │  → Orchestration, transactions
├─────────────────────────────────────┤
│  Domain (Entities, Value Objects)   │  → Business rules, invariants
├─────────────────────────────────────┤
│  Infrastructure (Repositories)      │  → Persistence, external services
└─────────────────────────────────────┘
```

```java
// Presentation: HTTP concerns only
@Path("/orders")
public class OrderResource {
    @Inject OrderService orderService;
    @Inject OrderMapper mapper;

    @POST
    public Response createOrder(@Valid CreateOrderRequest request) {
        Order order = orderService.createOrder(mapper.toCommand(request));
        return Response.created(buildLocation(order)).build();
    }
}

// Application: Use case orchestration
@ApplicationScoped
public class OrderService {
    @Inject OrderRepository repository;
    @Inject InventoryService inventory;
    @Inject PaymentService payment;

    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        inventory.reserve(command.getItems());
        Order order = Order.create(command);
        payment.authorize(order);
        return repository.save(order);
    }
}

// Domain: Business rules
public class Order {
    public static Order create(CreateOrderCommand command) {
        validateItems(command.getItems());
        return new Order(command.getCustomerId(), command.getItems());
    }

    private static void validateItems(List<OrderItem> items) {
        if (items.isEmpty()) {
            throw new EmptyOrderException();
        }
    }
}

// Infrastructure: Persistence
@ApplicationScoped
public class OrderRepository {
    @PersistenceContext EntityManager em;

    public Order save(Order order) {
        em.persist(order);
        return order;
    }
}
```

---

## Applying SRP to Methods

Methods should also have single responsibilities:

### Before - Method Doing Too Much

```java
public void processOrder(Order order) {
    // Validate
    if (order.getItems().isEmpty()) {
        throw new EmptyOrderException();
    }
    for (Item item : order.getItems()) {
        if (inventory.getStock(item.getSku()) < item.getQuantity()) {
            throw new InsufficientStockException(item);
        }
    }

    // Calculate
    Money subtotal = Money.ZERO;
    for (Item item : order.getItems()) {
        subtotal = subtotal.add(item.getPrice().multiply(item.getQuantity()));
    }
    Money tax = subtotal.multiply(TAX_RATE);
    Money total = subtotal.add(tax);
    order.setTotal(total);

    // Persist
    entityManager.persist(order);

    // Notify
    emailService.sendConfirmation(order);
}
```

### After - Separated Concerns

```java
public void processOrder(Order order) {
    validateOrder(order);
    calculateTotals(order);
    persistOrder(order);
    notifyCustomer(order);
}

private void validateOrder(Order order) {
    orderValidator.validate(order);
}

private void calculateTotals(Order order) {
    OrderPricing pricing = pricingService.calculate(order);
    order.applyPricing(pricing);
}

private void persistOrder(Order order) {
    orderRepository.save(order);
}

private void notifyCustomer(Order order) {
    notificationService.sendOrderConfirmation(order);
}
```

---

## Common SRP Violations in Jakarta EE

| Violation                             | Solution                                   |
| ------------------------------------- | ------------------------------------------ |
| Entity with business logic + JPA      | Separate domain model from JPA entity      |
| REST resource with validation + logic | Use Bean Validation + service layer        |
| Service with DB access + email        | Separate repository + notification service |
| DTO with mapping + validation         | Separate mapper class                      |

---

## Testing Benefits

SRP makes testing easier:

```java
// Easy to test in isolation
@Test
void shouldCalculateTotalCorrectly() {
    PricingService service = new PricingService();
    Order order = createOrderWithItems(/*...*/);

    OrderPricing pricing = service.calculate(order);

    assertThat(pricing.getSubtotal()).isEqualTo(Money.of(100));
    assertThat(pricing.getTax()).isEqualTo(Money.of(10));
}

// No need to mock database, email, etc.
```

---

## Related Recipes

- [Cohesion](./cohesion.md): Related fields and methods together
- [Small Classes](./small-classes.md): Keep classes focused
- [Feature Envy](./feature-envy.md): Move behavior to the right class
