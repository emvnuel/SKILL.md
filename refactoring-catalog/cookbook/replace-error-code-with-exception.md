# Replace Error Code with Exception

## Intent

Replace error codes with exceptions when an error is truly exceptional and shouldn't be handled by normal program flow.

## Code Smells That Indicate This Refactoring

- Functions return special values (-1, null, error codes) to indicate errors
- Callers must remember to check return values
- Error handling obscures the main logic
- Errors can be silently ignored

## Mechanics

1. Create an appropriate exception class if needed
2. Find all callers and adjust them to catch the exception
3. Change the body of the function to throw the exception instead of returning error code
4. Change the function's signature if needed (return type may change)
5. Test

## Example

### Before

```java
public class ShippingService {
    public int calculateShipping(Order order) {
        if (order.getCountry() == null) {
            return -23;  // Error code: missing country
        }
        if (!isValidCountry(order.getCountry())) {
            return -42;  // Error code: invalid country
        }
        return calculateRate(order);
    }
}

// Usage - easy to forget error checking
int rate = shippingService.calculateShipping(order);
if (rate < 0) {
    // Handle error, what does -23 mean again?
}
```

### After

```java
public class ShippingService {
    public int calculateShipping(Order order) {
        if (order.getCountry() == null) {
            throw new ShippingException("Country is required for shipping calculation");
        }
        if (!isValidCountry(order.getCountry())) {
            throw new InvalidCountryException(order.getCountry());
        }
        return calculateRate(order);
    }
}

public class ShippingException extends RuntimeException {
    public ShippingException(String message) {
        super(message);
    }
}

public class InvalidCountryException extends ShippingException {
    private final String country;

    public InvalidCountryException(String country) {
        super("Invalid country for shipping: " + country);
        this.country = country;
    }

    public String getCountry() {
        return country;
    }
}

// Usage - can't ignore the error
try {
    int rate = shippingService.calculateShipping(order);
    // Use rate
} catch (InvalidCountryException e) {
    showError("We don't ship to " + e.getCountry());
} catch (ShippingException e) {
    showError(e.getMessage());
}
```

### Optional for Expected Cases

When null or missing values are expected, use Optional instead:

```java
// Before - returns null
public User findUser(String id) {
    User user = userRepository.findById(id);
    if (user == null) {
        return null;  // Not found
    }
    return user;
}

// After - Optional for expected "not found" case
public Optional<User> findUser(String id) {
    return Optional.ofNullable(userRepository.findById(id));
}

// But throw exception for unexpected errors
public User getUser(String id) {
    return findUser(id)
        .orElseThrow(() -> new UserNotFoundException(id));
}
```

### Rich Exception

```java
public class ValidationException extends RuntimeException {
    private final List<ValidationError> errors;

    public ValidationException(List<ValidationError> errors) {
        super("Validation failed: " + errors.size() + " error(s)");
        this.errors = new ArrayList<>(errors);
    }

    public List<ValidationError> getErrors() {
        return Collections.unmodifiableList(errors);
    }
}

public record ValidationError(String field, String message) {}

// Usage
try {
    service.processOrder(order);
} catch (ValidationException e) {
    for (ValidationError error : e.getErrors()) {
        displayError(error.field(), error.message());
    }
}
```

## When to Use

✅ **Use Replace Error Code with Exception when:**

- The error is truly exceptional
- Error codes are easily ignored
- Error handling clutters normal flow
- Errors need rich context (stack trace, data)

❌ **Avoid Replace Error Code with Exception when:**

- The condition is expected and normal
- Performance is critical (exceptions are slow)
- Caller is expected to handle the case routinely
- Use Optional or Result types instead for expected cases

## Related Refactorings

- **Replace Exception with Precheck**: The inverse for expected conditions
- **Introduce Assertion**: For programmer errors
- **Introduce Special Case**: For null handling
