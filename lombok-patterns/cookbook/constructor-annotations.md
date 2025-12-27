# Constructor Annotations

## Overview

Lombok provides three annotations for constructor generation:

| Annotation                 | Generates Constructor For     |
| -------------------------- | ----------------------------- |
| `@NoArgsConstructor`       | No arguments                  |
| `@AllArgsConstructor`      | All fields                    |
| `@RequiredArgsConstructor` | `final` and `@NonNull` fields |

## @NoArgsConstructor

Creates a no-argument constructor.

```java
@NoArgsConstructor
public class User {
    private String name;
    private int age;
}

// Equivalent to:
public User() {}
```

### With Final Fields

```java
@NoArgsConstructor(force = true)  // Required for final fields
public class Config {
    private final String setting;  // Will be initialized to null/0/false
}
```

### Access Control

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // For JPA
@Entity
public class Product {
    @Id private Long id;
    private String name;
}
```

## @AllArgsConstructor

Creates a constructor with all fields as parameters.

```java
@AllArgsConstructor
public class Order {
    private String id;
    private String customer;
    private BigDecimal total;
}

// Equivalent to:
public Order(String id, String customer, BigDecimal total) {
    this.id = id;
    this.customer = customer;
    this.total = total;
}
```

### Static Factory Method

```java
@AllArgsConstructor(staticName = "of")
public class Point {
    private int x;
    private int y;
}

// Usage
Point p = Point.of(10, 20);  // Static factory instead of 'new'
```

## @RequiredArgsConstructor

Creates a constructor for `final` fields and `@NonNull` fields.

```java
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;  // Included
    private final PaymentService paymentService;    // Included
    private String lastOrderId;                     // Not included
}

// Equivalent to:
public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
    this.orderRepository = orderRepository;
    this.paymentService = paymentService;
}
```

### With @NonNull

```java
@RequiredArgsConstructor
public class Notification {
    private final String type;

    @NonNull
    private String message;  // Included (even though not final)

    private String details;  // Not included
}
```

## Common Patterns

### JPA Entity

```java
@Entity
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // JPA requirement
@AllArgsConstructor  // For programmatic creation
public class Customer {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private String email;
}
```

### Spring Service

```java
@Service
@RequiredArgsConstructor  // Constructor injection
@Slf4j
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final EmailService emailService;

    // Constructor automatically generated for all final fields
}
```

### Immutable DTO with Builder

```java
@Value  // All fields become final
@AllArgsConstructor  // Already included by @Value
@Builder
public class OrderDTO {
    String orderId;
    String customerName;
    List<ItemDTO> items;
    BigDecimal total;
}
```

## Combining Constructors

```java
@NoArgsConstructor
@AllArgsConstructor
public class Person {
    private String name;
    private int age;
}

// Generates both:
// public Person() {}
// public Person(String name, int age) {...}
```

## Pitfalls

| Issue                                    | Solution                                    |
| ---------------------------------------- | ------------------------------------------- |
| JPA needs no-arg constructor             | Add `@NoArgsConstructor`                    |
| `@Builder` without `@AllArgsConstructor` | Builder handles it, but explicit is clearer |
| Final fields with `@NoArgsConstructor`   | Use `force = true`                          |
| Multiple constructors conflict           | Use `@Builder` instead                      |

## Related Annotations

- **@Builder**: Fluent construction
- **@NonNull**: Mark required parameters
- **@Data**: Includes `@RequiredArgsConstructor`
