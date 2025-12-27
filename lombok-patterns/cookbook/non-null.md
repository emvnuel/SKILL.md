# @NonNull Annotation

## Intent

Generate null checks for parameters and fields to fail fast on null values.

## Basic Usage

### On Method Parameters

```java
public class UserService {
    public User createUser(@NonNull String name, @NonNull String email) {
        // Null check generated at start of method
        return new User(name, email);
    }
}

// Equivalent to:
public User createUser(String name, String email) {
    if (name == null) {
        throw new NullPointerException("name is marked non-null but is null");
    }
    if (email == null) {
        throw new NullPointerException("email is marked non-null but is null");
    }
    return new User(name, email);
}
```

### On Constructor Parameters

```java
@RequiredArgsConstructor
public class Order {
    @NonNull
    private final String orderId;

    @NonNull
    private final Customer customer;

    private String notes;  // Optional
}

// Generated constructor includes null checks
```

### On Fields

```java
public class Config {
    @NonNull
    private String host = "localhost";  // Cannot be set to null

    public void setHost(@NonNull String host) {
        this.host = host;  // Null check generated
    }
}
```

## With Other Annotations

### With @Data

```java
@Data
public class Person {
    @NonNull
    private String name;

    private Integer age;
}

// Setter for 'name' will include null check
// Constructor includes 'name' parameter with null check
```

### With @Builder

```java
@Builder
public class Email {
    @NonNull
    private String to;

    @NonNull
    private String subject;

    private String body;
}

// Builder will validate non-null fields on build()
```

## Difference from Java Annotations

| Annotation                               | Source          | Effect                    |
| ---------------------------------------- | --------------- | ------------------------- |
| `lombok.NonNull`                         | Lombok          | Generates null check code |
| `jakarta.validation.constraints.NotNull` | Bean Validation | Runtime validation        |
| `org.jetbrains.annotations.NotNull`      | IntelliJ        | IDE warnings only         |
| `java.util.Objects.requireNonNull`       | Java            | Manual null check         |

## Combining Approaches

```java
@Data
public class CreateUserRequest {
    @NonNull  // Lombok null check in setter/constructor
    @NotBlank // Bean Validation for API
    private String username;

    @NonNull
    @Email
    private String email;
}
```

## When to Use

✅ **Use `@NonNull` for:**

- Required constructor parameters
- Method parameters that must not be null
- Fail-fast validation
- Defensive programming

❌ **Avoid `@NonNull` for:**

- Optional fields
- Fields with valid null states
- When using Optional instead
- External API boundaries (use validation)

## Pitfalls

| Issue                                 | Solution                                      |
| ------------------------------------- | --------------------------------------------- |
| NPE instead of custom exception       | Handle validation separately                  |
| Confusion with validation annotations | Lombok is compile-time, validation is runtime |
| Not applied to getters                | Only affects setters and constructors         |

## Related Annotations

- **@RequiredArgsConstructor**: Includes @NonNull fields
- **@Data**: Generates setters with null checks
