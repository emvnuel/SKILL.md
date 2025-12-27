# Constructor-Based Mapping

## The Key Insight

> When using constructors, any field change causes a **compiler error**. With setters, you get a **runtime null**.

## Basic Pattern

```java
public class Customer {
    private final String id;
    private final String name;
    private final String email;

    // Single constructor - MapStruct uses this automatically
    public Customer(String id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }
}

@Mapper(componentModel = "cdi")
public interface CustomerMapper {
    Customer toEntity(CreateCustomerRequest request);
}
```

Generated code uses constructor:

```java
public Customer toEntity(CreateCustomerRequest request) {
    if (request == null) return null;

    return new Customer(
        request.getId(),
        request.getName(),
        request.getEmail()
    );  // Constructor call = compile-safe
}
```

## Why This Matters

### With Setters (Dangerous)

```java
public class OrderDTO {
    private String id;
    private String status;
    // Later: Add this field
    private String customerName;

    // Setters...
}

// Generated code calls setters:
public OrderDTO toDto(Order order) {
    OrderDTO dto = new OrderDTO();
    dto.setId(order.getId());
    dto.setStatus(order.getStatus());
    // customerName is NEVER SET - null at runtime!
    return dto;
}
```

### With Constructors (Safe)

```java
public record OrderDTO(String id, String status, String customerName) {}

// Won't compile until you add mapping:
// "No property named 'customerName' exists in source"
```

## Parameter Name Matching

MapStruct matches constructor parameters by name:

```java
// Source
public record CreateOrderRequest(String customerId, BigDecimal amount) {}

// Target
public class Order {
    @Default
    public Order(String customerId, BigDecimal amount) {
        // Parameter names match - works automatically
    }
}

@Mapper(componentModel = "cdi")
public interface OrderMapper {
    Order toEntity(CreateOrderRequest request);  // Just works
}
```

## When Names Don't Match

Use `@Mapping`:

```java
// Source has 'userId', target constructor has 'customerId'
@Mapper(componentModel = "cdi")
public interface OrderMapper {

    @Mapping(target = "customerId", source = "userId")
    Order toEntity(CreateOrderRequest request);
}
```

## Complex Mappings

```java
@Mapper(componentModel = "cdi")
public interface OrderMapper {

    @Mapping(target = "orderId", source = "id")
    @Mapping(target = "totalFormatted", expression = "java(formatMoney(order.getTotal()))")
    @Mapping(target = "status", source = "status.displayName")
    OrderResponse toResponse(Order order);

    default String formatMoney(BigDecimal amount) {
        return NumberFormat.getCurrencyInstance().format(amount);
    }
}
```

## Ignore Fields

For fields set by entity logic:

```java
@Entity
public class Order {
    @Default
    public Order(String customerId, BigDecimal total) {
        this.customerId = customerId;
        this.total = total;
        this.createdAt = Instant.now();  // Set internally
    }
}

@Mapper(componentModel = "cdi")
public interface OrderMapper {

    @Mapping(target = "createdAt", ignore = true)  // Not from source
    Order toEntity(CreateOrderRequest request);
}
```

## Related Entries

- [record-mapping](record-mapping.md) - Java Records
- [entity-mapping](entity-mapping.md) - JPA entities
