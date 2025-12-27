# @With Annotation

## Intent

Generate "wither" methods that create a new instance with one field changed — essential for immutable objects.

## Basic Usage

```java
@Value
public class Person {
    String name;
    int age;

    @With
    String email;
}

// Usage
Person p1 = new Person("John", 30, "john@example.com");
Person p2 = p1.withEmail("newemail@example.com");

// p1 is unchanged: Person(name=John, age=30, email=john@example.com)
// p2 is new instance: Person(name=John, age=30, email=newemail@example.com)
```

## On All Fields

```java
@With
@AllArgsConstructor
public class Point {
    private final int x;
    private final int y;
}

// Generates:
// public Point withX(int x) { return new Point(x, this.y); }
// public Point withY(int y) { return new Point(this.x, y); }

Point p1 = new Point(10, 20);
Point p2 = p1.withX(15).withY(25);  // Chaining
```

## With @Value (Recommended Pattern)

```java
@Value
@With
public class ImmutableConfig {
    String host;
    int port;
    boolean ssl;
    Duration timeout;
}

// Create base and derive variations
ImmutableConfig base = new ImmutableConfig("localhost", 8080, false, Duration.ofSeconds(30));
ImmutableConfig prod = base.withHost("prod.example.com").withSsl(true);
ImmutableConfig dev = base.withPort(9090);
```

## Access Level Control

```java
@Value
public class Account {
    String id;

    @With(AccessLevel.PACKAGE)
    BigDecimal balance;
}

// withBalance() is package-private, not public
```

## When to Use

✅ **Use `@With` for:**

- Immutable value objects
- Creating modified copies
- State transitions in immutable style
- Configuration objects

❌ **Avoid `@With` for:**

- Mutable objects (just use setters)
- Objects with many interdependent fields
- When modification should be restricted

## Comparison with Setters

| Setter                   | Wither              |
| ------------------------ | ------------------- |
| Modifies existing object | Creates new object  |
| `void setX(value)`       | `Type withX(value)` |
| Mutable pattern          | Immutable pattern   |
| `@Setter`                | `@With`             |

## Requirements

`@With` requires:

- An all-args constructor (explicit or via `@AllArgsConstructor`/`@Value`)
- Fields in the same order as constructor parameters

```java
// ✅ Works - @Value provides all-args constructor
@Value
@With
public class Order {
    String id;
    BigDecimal total;
}

// ✅ Works - explicit constructor
@With
public class Order {
    private final String id;
    private final BigDecimal total;

    public Order(String id, BigDecimal total) {
        this.id = id;
        this.total = total;
    }
}
```

## Related Annotations

- **@Value**: Natural companion for immutability
- **@Builder(toBuilder = true)**: Alternative for bulk modifications
