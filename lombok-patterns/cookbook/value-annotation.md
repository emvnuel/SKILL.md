# @Value Annotation

> ⚠️ **Java 21+ Note**: For simple immutable value objects, prefer **Java Records** instead of `@Value`. Records are built into the language and provide the same benefits with less dependency.

## Intent

Create immutable classes with getters, `toString()`, `equals()`, and `hashCode()` — all fields become `private final`.

## What It Generates

`@Value` is equivalent to:

- Making the class `final`
- Making all fields `private final`
- `@Getter` on all fields (no setters!)
- `@AllArgsConstructor`
- `@ToString`
- `@EqualsAndHashCode`

## Java Record Alternative (Preferred)

```java
// ✅ Prefer Java Record for simple value objects (Java 16+)
public record Money(BigDecimal amount, String currency) {
    // Compact constructor for validation
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
    }

    // Additional methods
    public Money add(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(amount.add(other.amount), currency);
    }
}
```

## When to Still Use @Value

```java
// ✅ Use @Value with @Builder (records don't have builder)
@Value
@Builder
public class EmailMessage {
    String to;
    String subject;
    String body;
    @Builder.Default
    LocalDateTime timestamp = LocalDateTime.now();
}

// ✅ Use @Value with inheritance (records can't extend)
@Value
public class ExtendedConfig extends BaseConfig {
    String additionalSetting;
}

// ✅ Use @Value for complex defaults
@Value
public class ComplexObject {
    String name;
    @With
    Map<String, Object> properties;
}
```

## When to Use

✅ **Use `@Value` for:**

- Immutable classes needing `@Builder`
- Classes with inheritance requirements
- Complex objects with `@With` for cloning
- Legacy codebases before Java 16

❌ **Avoid `@Value` for:**

- Simple value objects → use **Java Records**
- Simple DTOs → use **Java Records**
- JPA entities (need no-arg constructor, mutable state)

## Creating Modified Copies

### With Java Record

```java
public record Person(String name, int age, String email) {
    public Person withEmail(String newEmail) {
        return new Person(name, age, newEmail);
    }
}
```

### With @Value and @With

```java
@Value
public class Person {
    String name;
    int age;
    @With String email;
}

Person p1 = new Person("John", 30, "john@example.com");
Person p2 = p1.withEmail("newemail@example.com");
```

## Pitfalls

| Issue                   | Solution                               |
| ----------------------- | -------------------------------------- |
| Simple value objects    | Use Java Records instead               |
| Need no-arg constructor | Add `@NoArgsConstructor(force = true)` |
| Need inheritance        | Use `@Value` (records can't extend)    |
| Need builder            | Use `@Value` + `@Builder`              |

## Related Annotations

- **Java Record**: Preferred for simple immutable objects (Java 16+)
- **@Data**: For mutable classes
- **@With**: For creating modified copies
- **@Builder**: For fluent construction
