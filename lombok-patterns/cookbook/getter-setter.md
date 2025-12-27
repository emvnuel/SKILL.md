# @Getter and @Setter Annotations

## Intent

Generate getter and setter methods for fields without manual boilerplate.

## Basic Usage

### On Class

```java
@Getter
@Setter
public class Person {
    private String name;
    private int age;
    private String email;
}
// Generates: getName(), setName(), getAge(), setAge(), getEmail(), setEmail()
```

### On Individual Fields

```java
public class Account {
    @Getter
    private final String id;  // Only getter (no setter for final)

    @Getter @Setter
    private String name;

    @Getter
    private double balance;  // Read-only externally

    void deposit(double amount) {
        this.balance += amount;  // Internal modification allowed
    }
}
```

## Access Level Control

```java
public class User {
    @Getter
    @Setter(AccessLevel.PROTECTED)
    private String internalId;

    @Getter(AccessLevel.PACKAGE)
    @Setter(AccessLevel.PRIVATE)
    private String secretKey;

    @Getter(AccessLevel.NONE)  // Suppress getter
    private String hiddenField;
}
```

### Access Levels

| Level       | Visibility                |
| ----------- | ------------------------- |
| `PUBLIC`    | Default                   |
| `PROTECTED` | Same package + subclasses |
| `PACKAGE`   | Same package only         |
| `PRIVATE`   | Same class only           |
| `NONE`      | Don't generate            |

## Boolean Fields

```java
public class Settings {
    @Getter @Setter
    private boolean enabled;  // isEnabled(), setEnabled()

    @Getter @Setter
    private Boolean visible;  // getVisible(), setVisible()
}
```

## Lazy Getter

```java
public class ExpensiveResource {
    @Getter(lazy = true)
    private final BigDecimal cachedValue = computeExpensiveValue();

    private BigDecimal computeExpensiveValue() {
        // This runs only on first access
        return new BigDecimal("42");
    }
}
```

## When to Use

✅ **Use `@Getter`/`@Setter` when:**

- You need fine-grained control over access
- Different access levels for different fields
- Mixing with JPA entities
- Some fields should be read-only

❌ **Avoid when:**

- All fields need same access → use `@Data` or `@Value`
- Encapsulation violation (exposing internal state)

## JPA Entity Pattern

```java
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Product {
    @Id
    @Setter(AccessLevel.NONE)  // ID shouldn't be settable
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Setter(AccessLevel.PROTECTED)  // Control relationship modification
    @ManyToOne
    private Category category;
}
```

## Custom Getter/Setter Names

For non-standard naming (e.g., fluent accessors):

```java
@Accessors(fluent = true)
@Getter
@Setter
public class Point {
    private int x;
    private int y;
}

// Usage
Point p = new Point();
p.x(10).y(20);  // Fluent setters return 'this'
int x = p.x();  // Fluent getters
```

## Related Annotations

- **@Data**: Includes both @Getter and @Setter
- **@Value**: Includes only @Getter (immutable)
- **@Accessors**: Customize accessor behavior
