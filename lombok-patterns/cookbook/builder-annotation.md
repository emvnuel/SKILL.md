# @Builder Annotation

## Intent

Generate a builder pattern implementation for fluent, readable object construction.

## Basic Usage

```java
@Builder
public class Email {
    private String to;
    private String from;
    private String subject;
    private String body;
}

// Usage
Email email = Email.builder()
    .to("user@example.com")
    .from("noreply@company.com")
    .subject("Welcome!")
    .body("Hello and welcome...")
    .build();
```

## @Builder.Default

Set default values for fields:

```java
@Builder
public class HttpRequest {
    private String url;
    private String method;

    @Builder.Default
    private int timeout = 30000;

    @Builder.Default
    private Map<String, String> headers = new HashMap<>();
}

// timeout and headers have defaults if not specified
HttpRequest req = HttpRequest.builder()
    .url("https://api.example.com")
    .method("GET")
    .build();
```

## @Singular for Collections

Handle collections elegantly:

```java
@Builder
public class Team {
    private String name;

    @Singular
    private List<String> members;

    @Singular("skill")
    private Set<String> skills;
}

// Add items one by one
Team team = Team.builder()
    .name("Engineering")
    .member("Alice")
    .member("Bob")
    .member("Charlie")
    .skill("Java")
    .skill("Spring")
    .build();

// Or all at once
Team team2 = Team.builder()
    .name("Design")
    .members(List.of("Diana", "Eve"))
    .build();
```

## Combining with Other Annotations

### With @Value (Immutable)

```java
@Value
@Builder
public class OrderDTO {
    String orderId;
    String customer;
    BigDecimal total;
}
```

### With @Data and Constructors

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserDTO {
    private String name;
    private String email;
}
```

## @SuperBuilder for Inheritance

```java
@SuperBuilder
public abstract class Vehicle {
    private String brand;
    private String model;
}

@SuperBuilder
public class Car extends Vehicle {
    private int doors;
    private boolean sunroof;
}

// Usage
Car car = Car.builder()
    .brand("Toyota")
    .model("Camry")
    .doors(4)
    .sunroof(true)
    .build();
```

## toBuilder() - Create Modified Copies

```java
@Builder(toBuilder = true)
public class Config {
    private String host;
    private int port;
    private boolean ssl;
}

// Create modified copy
Config prod = Config.builder()
    .host("localhost")
    .port(8080)
    .ssl(false)
    .build();

Config staging = prod.toBuilder()
    .host("staging.example.com")
    .ssl(true)
    .build();
```

## Builder Method on Existing Class

```java
public class Person {
    private String firstName;
    private String lastName;
    private int age;

    @Builder
    public Person(String firstName, String lastName, int age) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.age = age;
    }
}
```

## When to Use

✅ **Use `@Builder` for:**

- Classes with many fields
- Optional fields with sensible defaults
- Immutable objects (combine with `@Value`)
- Complex object construction

❌ **Avoid `@Builder` for:**

- Simple classes with few fields
- Classes that need no-arg constructor for frameworks
- When regular constructors are clearer

## Pitfalls

| Issue                                 | Solution                              |
| ------------------------------------- | ------------------------------------- |
| JPA needs no-arg constructor          | Add `@NoArgsConstructor`              |
| Builder doesn't include parent fields | Use `@SuperBuilder`                   |
| Default values not working            | Use `@Builder.Default`                |
| Collection NPE                        | Use `@Singular` or `@Builder.Default` |

## Related Annotations

- **@Value**: Immutable classes
- **@SuperBuilder**: Inheritance support
- **@Singular**: Collection handling
