# Bean Validation Patterns with CDD

## Principle

Use Bean Validation for boundary protection and explicit business rule validation.

## Basic DTO Validation

```java
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be 2-100 characters")
    String name,

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email,

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    @Pattern(regexp = ".*[A-Z].*", message = "Password must contain uppercase")
    @Pattern(regexp = ".*[0-9].*", message = "Password must contain digit")
    String password
) {}
```

## Cross-Field Validation

```java
@ValidDateRange  // Custom constraint
public record DateRangeRequest(
    @NotNull LocalDate startDate,
    @NotNull LocalDate endDate
) {}

// Constraint definition
@Target({TYPE})
@Retention(RUNTIME)
@Constraint(validatedBy = DateRangeValidator.class)
public @interface ValidDateRange {
    String message() default "End date must be after start date";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Validator
public class DateRangeValidator
    implements ConstraintValidator<ValidDateRange, DateRangeRequest> {

    @Override
    public boolean isValid(DateRangeRequest value, ConstraintValidatorContext ctx) {
        if (value.startDate() == null || value.endDate() == null) {
            return true;  // @NotNull handles null
        }
        return value.endDate().isAfter(value.startDate());
    }
}
```

## Business Rule Validators

```java
// Domain-specific validation
@Target({FIELD, PARAMETER})
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
    @UniqueEmail
    @Email
    String email
) {}
```

## Validation Groups

```java
public interface OnCreate {}
public interface OnUpdate {}

public record ProductRequest(
    @Null(groups = OnCreate.class, message = "ID must be null for creation")
    @NotNull(groups = OnUpdate.class, message = "ID required for update")
    Long id,

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    String name,

    @NotNull(groups = {OnCreate.class, OnUpdate.class})
    @Positive
    BigDecimal price
) {}

// In resource
@POST
public Response create(
        @Valid @ConvertGroup(to = OnCreate.class) ProductRequest request) {
    // ...
}

@PUT
@Path("/{id}")
public Response update(
        @PathParam("id") Long id,
        @Valid @ConvertGroup(to = OnUpdate.class) ProductRequest request) {
    // ...
}
```

## Manual Validation in Service

```java
@ApplicationScoped
public class OrderService {

    @Inject
    private Validator validator;

    public Order create(CreateOrderRequest request) {
        // Validate with specific group
        Set<ConstraintViolation<CreateOrderRequest>> violations =
            validator.validate(request, OnCreate.class);

        if (!violations.isEmpty()) {
            throw new ConstraintViolationException(violations);
        }

        // Proceed with valid request
        return /* ... */;
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
        List<ValidationError> errors = e.getConstraintViolations().stream()
            .map(v -> new ValidationError(
                extractFieldName(v.getPropertyPath()),
                v.getMessage()))
            .toList();

        return Response.status(Response.Status.BAD_REQUEST)
            .entity(new ValidationErrorResponse(errors))
            .build();
    }

    private String extractFieldName(Path path) {
        String fullPath = path.toString();
        int dotIndex = fullPath.lastIndexOf('.');
        return dotIndex > 0 ? fullPath.substring(dotIndex + 1) : fullPath;
    }
}

public record ValidationError(String field, String message) {}
public record ValidationErrorResponse(List<ValidationError> errors) {}
```

## Entity Validation

```java
@Entity
public class Product {

    @NotBlank
    @Size(max = 100)
    private String name;

    @NotNull
    @Positive
    private BigDecimal price;

    @Size(max = 1000)
    private String description;

    // JPA validates on persist/update if validation-mode is set
}
```

## Related Entries

- [protect-boundaries](protect-boundaries.md) - Boundary protection
- [boundary-separation](boundary-separation.md) - DTO patterns
- [explicit-business-rules](explicit-business-rules.md) - Business validation
