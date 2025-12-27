# Custom @Default Annotation

## The Problem

MapStruct defaults to parameterless constructors. For mutable entities with multiple constructors, you need to tell MapStruct which one to use.

## Solution: Create Your Own @Default

```java
package com.example.mapstruct;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a constructor as the default for MapStruct mapping.
 * MapStruct recognizes any annotation named "Default" from any package.
 */
@Target(ElementType.CONSTRUCTOR)
@Retention(RetentionPolicy.CLASS)
public @interface Default {
}
```

## Usage on Entities

```java
@Entity
public class Order {

    @Id @GeneratedValue
    private Long id;

    private String customerId;
    private BigDecimal total;
    private OrderStatus status;
    private Instant createdAt;

    // JPA requires no-arg constructor
    protected Order() {}

    // MapStruct uses this constructor
    @Default
    public Order(String customerId, BigDecimal total, OrderStatus status) {
        this.customerId = customerId;
        this.total = total;
        this.status = status;
        this.createdAt = Instant.now();
    }

    // Business constructor - not used by MapStruct
    public Order(String customerId, List<OrderItem> items) {
        this(customerId, calculateTotal(items), OrderStatus.PENDING);
    }
}
```

## How It Works

MapStruct uses reflection at compile time to find annotations. It matches by **simple name**:

```java
// All of these work:
@com.example.mapstruct.Default
@org.example.Default
@lombok.experimental.Default
// Any annotation named "Default"
```

## Constructor Selection Priority

1. Constructor annotated with `@Default`
2. Single public constructor
3. Parameterless constructor
4. Compilation error (ambiguous)

```java
public class Product {

    // ❌ Ambiguous - will cause compilation error
    public Product(String name) {}
    public Product(String name, String category) {}
}

public class Product {

    // ✓ Clear - @Default resolves ambiguity
    public Product(String name) {}

    @Default
    public Product(String name, String category) {}
}
```

## Compile-Time Safety

When you change the constructor, MapStruct fails to compile:

```java
// Before
@Default
public Order(String customerId, BigDecimal total) { }

// After - added field
@Default
public Order(String customerId, BigDecimal total, String notes) { }
// Mapper now fails: "No property named 'notes' exists in source"
```

## With @ConstructorProperties

For parameter name resolution:

```java
@Default
@ConstructorProperties({"customerId", "total", "status"})
public Order(String customerId, BigDecimal total, OrderStatus status) {
    // ...
}
```

## Related Entries

- [constructor-mapping](constructor-mapping.md) - Using constructors
- [entity-mapping](entity-mapping.md) - JPA entity patterns
