# Manual Fallback Pattern

## The Idea

Use MapStruct to **generate** the initial mapping code, then copy and maintain it manually.

> "Think of it as a crappy auto-complete" — agentgt

## When to Use

- JSpecify/nullability annotations cause issues
- Need precise control over mapping logic
- One-time generation, then manual maintenance
- AI-assisted coding makes this less necessary now

## Step 1: Create the Mapper

```java
@Mapper(componentModel = "cdi")
public interface OrderMapper {

    @Mapping(target = "orderId", source = "id")
    @Mapping(target = "total", expression = "java(order.getTotal().toString())")
    OrderResponse toResponse(Order order);
}
```

## Step 2: Build to Generate

```bash
mvn compile
```

## Step 3: Copy Generated Code

From `target/generated-sources/annotations/`:

```java
// target/generated-sources/annotations/com/example/OrderMapperImpl.java
@ApplicationScoped
public class OrderMapperImpl implements OrderMapper {

    @Override
    public OrderResponse toResponse(Order order) {
        if (order == null) {
            return null;
        }

        return new OrderResponse(
            order.getId().toString(),
            order.getCustomerId(),
            order.getTotal().toString(),
            order.getStatus().name()
        );
    }
}
```

## Step 4: Make It Manual

```java
// src/main/java/com/example/OrderMapperImpl.java
@ApplicationScoped
public class OrderMapperImpl implements OrderMapper {

    @Override
    public OrderResponse toResponse(Order order) {
        if (order == null) {
            return null;
        }

        // Now you can customize freely
        return new OrderResponse(
            order.getId().toString(),
            order.getCustomerId(),
            formatMoney(order.getTotal()),  // Custom formatting
            translateStatus(order.getStatus())  // Custom logic
        );
    }

    private String formatMoney(BigDecimal amount) {
        return NumberFormat.getCurrencyInstance().format(amount);
    }

    private String translateStatus(OrderStatus status) {
        return switch (status) {
            case PENDING -> "Aguardando";
            case CONFIRMED -> "Confirmado";
            case SHIPPED -> "Enviado";
        };
    }
}
```

## Step 5: Disable Generation

Comment out or remove the original interface:

```java
// No longer generating
// @Mapper(componentModel = "cdi")
public interface OrderMapper {
    OrderResponse toResponse(Order order);
}
```

## Why Constructor Mapping Still Helps

Even in manual code, using constructors = compile-time safety:

```java
// If OrderResponse adds a field, this FAILS TO COMPILE
return new OrderResponse(
    order.getId().toString(),
    order.getCustomerId(),
    order.getTotal().toString(),
    order.getStatus().name()
    // Missing new field = compiler error!
);
```

## Regenerate When Needed

1. Uncomment the `@Mapper` annotation
2. Build
3. Copy new generated code
4. Apply your customizations again
5. Comment out `@Mapper`

## Trade-offs

| Aspect        | Generated             | Manual                |
| ------------- | --------------------- | --------------------- |
| Maintenance   | Automatic             | Manual + compile-safe |
| Customization | Limited               | Full control          |
| Nullability   | Issues with JSpecify  | You decide            |
| Build time    | Annotation processing | None                  |

## Related Entries

- [constructor-mapping](constructor-mapping.md) - Why constructors matter
- [record-mapping](record-mapping.md) - Records for safety
