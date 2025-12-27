# Java Records Mapping

## Why Records Are Ideal

Records **only have constructors** — no setters. MapStruct automatically uses constructor mapping.

## Basic Record Mapping

```java
// Source
@Entity
public class Order {
    @Id private Long id;
    private String customerId;
    private BigDecimal total;
    private OrderStatus status;
    // getters...
}

// Target - Record
public record OrderResponse(
    String orderId,
    String customerId,
    String total,
    String status
) {}

@Mapper(componentModel = "cdi")
public interface OrderMapper {

    @Mapping(target = "orderId", source = "id")
    @Mapping(target = "total", expression = "java(order.getTotal().toString())")
    @Mapping(target = "status", expression = "java(order.getStatus().name())")
    OrderResponse toResponse(Order order);
}
```

## Nested Records

```java
public record AddressResponse(
    String street,
    String city,
    String zipCode
) {}

public record CustomerResponse(
    String id,
    String name,
    AddressResponse address
) {}

@Mapper(componentModel = "cdi")
public interface CustomerMapper {

    CustomerResponse toResponse(Customer customer);
    AddressResponse toResponse(Address address);
    // Nested mapping automatic when method exists
}
```

## Record to Entity

```java
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotEmpty List<OrderItemRequest> items
) {}

@Mapper(componentModel = "cdi", uses = OrderItemMapper.class)
public interface OrderMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "status", constant = "PENDING")
    Order toEntity(CreateOrderRequest request);
}
```

## Collections of Records

```java
public record OrderItemResponse(
    String productId,
    int quantity,
    String subtotal
) {}

public record OrderResponse(
    String orderId,
    List<OrderItemResponse> items,  // Nested collection
    String total
) {}

@Mapper(componentModel = "cdi")
public interface OrderMapper {

    OrderResponse toResponse(Order order);
    OrderItemResponse toResponse(OrderItem item);

    // Collection mapping is automatic
    List<OrderItemResponse> toResponseList(List<OrderItem> items);
}
```

## Compile-Time Safety Demo

```java
// Version 1
public record OrderResponse(String orderId, String status) {}

// Version 2 - Added field
public record OrderResponse(String orderId, String status, String customerName) {}

// Mapper now FAILS TO COMPILE:
// "Unmapped target property: 'customerName'"

// Fix:
@Mapping(target = "customerName", source = "customer.name")
OrderResponse toResponse(Order order);
```

## Record with Compact Constructor

```java
public record MoneyResponse(BigDecimal amount, String currency) {
    // Compact constructor for validation - still works with MapStruct
    public MoneyResponse {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Negative amount");
        }
    }
}
```

## Related Entries

- [constructor-mapping](constructor-mapping.md) - Why constructors matter
- [entity-mapping](entity-mapping.md) - JPA entity patterns
