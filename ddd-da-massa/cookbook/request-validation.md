# Request Validation Patterns

## Principle

> Validation belongs in DTOs, enforced by Bean Validation and the framework.

## Basic Validation

```java
public record CreateProductRequest(
    @NotBlank(message = "Name is required")
    @Size(max = 100)
    String name,

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be positive")
    BigDecimal price,

    @Size(max = 1000)
    String description
) {}
```

## Cross-Field Validation

Use compact constructor:

```java
public record DateRangeRequest(
    @NotNull LocalDate startDate,
    @NotNull LocalDate endDate
) {
    public DateRangeRequest {
        if (startDate != null && endDate != null
            && endDate.isBefore(startDate)) {
            throw new IllegalArgumentException(
                "End date must be after start date");
        }
    }
}
```

Or custom constraint:

```java
@ValidDateRange
public record DateRangeRequest(
    @NotNull LocalDate startDate,
    @NotNull LocalDate endDate
) {}

@Target(TYPE)
@Retention(RUNTIME)
@Constraint(validatedBy = DateRangeValidator.class)
public @interface ValidDateRange {
    String message() default "End date must be after start date";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

## Nested Validation

```java
public record CreateOrderRequest(
    @NotBlank String customerId,

    @NotEmpty(message = "Order must have items")
    @Valid  // Validates each item
    List<OrderItemRequest> items,

    @Valid  // Validates nested object
    AddressRequest shippingAddress
) {}

public record OrderItemRequest(
    @NotBlank String productId,
    @Positive int quantity
) {}
```

## Business Rule Validation with CDI

```java
@Target(FIELD)
@Retention(RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already registered";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@ApplicationScoped
public class UniqueEmailValidator
    implements ConstraintValidator<UniqueEmail, String> {

    @Inject
    private UserRepository users;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext ctx) {
        if (email == null) return true;
        return !users.existsByEmail(email);
    }
}

// Usage
public record CreateUserRequest(
    @NotBlank String name,

    @NotBlank
    @Email
    @UniqueEmail  // Custom business validation
    String email
) {}
```

## Resource with Validation

```java
@Path("/users")
@RequestScoped
public class UserResource {

    @Inject private UserRepository users;

    @POST
    @Transactional
    public Response create(@Valid CreateUserRequest request) {
        // If we get here, validation passed!
        User user = request.toEntity();
        users.save(user);
        return Response.created(/*...*/).build();
    }
}
```

## Exception Mapper

```java
@Provider
public class ValidationExceptionMapper
    implements ExceptionMapper<ConstraintViolationException> {

    @Override
    public Response toResponse(ConstraintViolationException e) {
        List<FieldError> errors = e.getConstraintViolations().stream()
            .map(v -> new FieldError(
                extractField(v.getPropertyPath()),
                v.getMessage()))
            .toList();

        return Response.status(Response.Status.BAD_REQUEST)
            .entity(new ValidationErrorResponse(errors))
            .build();
    }
}

public record FieldError(String field, String message) {}
public record ValidationErrorResponse(List<FieldError> errors) {}
```

## Cognitive Load: Zero Extra Points

Validation annotations don't add cognitive load — they're declarative metadata handled by the framework.

## Related Entries

- [form-value-objects](form-value-objects.md) - Smart DTOs
- [cohesive-resources](cohesive-resources.md) - Resource patterns
