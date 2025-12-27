# Form Value Objects (Smart DTOs)

## Principle

> DTOs should know how to convert themselves to domain objects. No separate Mapper classes needed.

## Problem

Separate mapper/converter classes add cognitive load without adding value.

```java
// ❌ Unnecessary indirection
public record CreateUserRequest(String name, String email) {}

// Separate mapper class
public class UserMapper {
    public static User toEntity(CreateUserRequest request) {
        return new User(request.name(), request.email());
    }
}

// Controller
User user = UserMapper.toEntity(request);  // +1 point for UserMapper
```

## Solution

DTOs convert themselves:

```java
public record CreateUserRequest(
    @NotBlank String name,
    @NotBlank @Email String email
) {
    public User toEntity() {
        return new User(name, email);
    }
}

// Controller
User user = request.toEntity();  // No extra class needed
```

## Complex Conversions

When conversion needs dependencies:

```java
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<@Valid OrderItemRequest> items
) {
    // Pass required dependencies as parameters
    public Order toEntity(Customer customer) {
        return new Order(
            customer,
            items.stream().map(OrderItemRequest::toEntity).toList()
        );
    }
}

public record OrderItemRequest(
    @NotBlank String productId,
    @Positive int quantity
) {
    public OrderItem toEntity() {
        return new OrderItem(productId, quantity);
    }
}
```

## Resource Usage

```java
@Path("/orders")
@RequestScoped
public class OrderResource {

    @Inject private CustomerRepository customers;
    @Inject private OrderRepository orders;

    @POST
    @Transactional
    public Response create(@Valid CreateOrderRequest request) {
        Customer customer = customers.findById(request.customerId())
            .orElseThrow(CustomerNotFoundException::new);

        Order order = request.toEntity(customer);
        orders.save(order);

        return Response.created(/*...*/).build();
    }
}
// No mapper classes = lower cognitive load
```

## With Repository Access in DTO

For complex forms that need lookups:

```java
public record CreateReviewRequest(
    Long goalId,
    String positivePoints,
    String improvingPoints
) {
    // Repository passed as parameter
    public PerformanceReview toEntity(GoalRepository goals, Employee employee) {
        Goal goal = goals.findById(goalId)
            .orElseThrow(GoalNotFoundException::new);
        return new PerformanceReview(employee, goal, positivePoints, improvingPoints);
    }
}
```

## Validation in DTO

```java
public record TransferRequest(
    @NotBlank String fromAccountId,
    @NotBlank String toAccountId,
    @NotNull @Positive BigDecimal amount
) {
    // Compact constructor for cross-field validation
    public TransferRequest {
        if (fromAccountId.equals(toAccountId)) {
            throw new IllegalArgumentException(
                "Source and destination must be different");
        }
    }

    public Transfer toEntity(Account from, Account to) {
        return new Transfer(from, to, amount);
    }
}
```

## Cognitive Load Impact

| Approach          | Points                |
| ----------------- | --------------------- |
| Mapper class      | +1 (extra class)      |
| DTO with toEntity | +0 (just method call) |

**Savings**: 1 point per conversion = significant in complex flows.

## Related Entries

- [request-validation](request-validation.md) - Validation in DTOs
- [rich-entities](rich-entities.md) - Where logic ends up
