# @Data Annotation

> ⚠️ **Java 21+ Note**: For simple DTOs without business logic, prefer **Java Records** instead of `@Data`. Records provide immutability and less boilerplate.

## Intent

Generate getters, setters, `toString()`, `equals()`, and `hashCode()` methods automatically with a single annotation.

## What It Generates

`@Data` is equivalent to:

- `@Getter` on all fields
- `@Setter` on all non-final fields
- `@ToString`
- `@EqualsAndHashCode`
- `@RequiredArgsConstructor`

## Java Record Alternative (Preferred for Simple Cases)

```java
// ✅ Prefer Java Record for simple DTOs (Java 16+)
public record UserDTO(Long id, String name, String email) {}

// Usage
UserDTO user = new UserDTO(1L, "John", "john@example.com");
String name = user.name();  // Accessor methods generated
```

## When to Still Use @Data

```java
// ✅ Use @Data when you need mutability
@Data
public class MutableUser {
    private Long id;
    private String name;
    private String email;
}

// ✅ Use @Data with inheritance (records can't extend classes)
@Data
public class Employee extends Person {
    private String department;
}
```

## When to Use

✅ **Use `@Data` for:**

- Mutable classes that need setters
- Classes with inheritance
- Classes requiring Jackson/JPA with no-arg constructor (with `@NoArgsConstructor`)
- Legacy codebases before Java 16

❌ **Avoid `@Data` for:**

- Simple DTOs → use **Java Records**
- Immutable objects → use **Java Records** or `@Value`
- JPA entities → use individual annotations
- Classes with sensitive fields (passwords in toString)

## Customization

### Exclude Fields from ToString

```java
@Data
public class User {
    private String username;

    @ToString.Exclude
    private String password;
}
```

### Exclude Fields from EqualsAndHashCode

```java
@Data
@EqualsAndHashCode(exclude = {"lastModified"})
public class Document {
    private String id;
    private String content;
    private LocalDateTime lastModified;
}
```

## Pitfalls

| Issue          | Problem                                  | Solution                                 |
| -------------- | ---------------------------------------- | ---------------------------------------- |
| Simple DTOs    | Unnecessary when Record suffices         | Use Java Records instead                 |
| JPA entities   | Lazy loading issues with equals/hashCode | Use `@Getter`, `@Setter` separately      |
| Mutable fields | All non-final fields get setters         | Use Records or `@Value` for immutability |
| Sensitive data | Passwords in toString()                  | Use `@ToString.Exclude`                  |

## Related Annotations

- **Java Record**: Preferred for simple value objects (Java 16+)
- **@Value**: For immutable classes (when Records don't fit)
- **@Getter/@Setter**: For selective access
- **@ToString**: For custom string representation
