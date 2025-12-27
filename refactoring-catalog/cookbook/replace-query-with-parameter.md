# Replace Query with Parameter

## Intent

Replace an internal query with a parameter, moving the responsibility for the query to the caller.

## Code Smells That Indicate This Refactoring

- Function has a hidden dependency (global, singleton, environment)
- Function is hard to test due to internal queries
- Want to make a function more pure/functional
- Need to break a dependency for modularity

## Mechanics

1. Apply **Extract Variable** on the query code
2. Apply **Extract Function** on the body code that doesn't use the query
3. Use **Inline Variable** to remove the variable
4. Apply **Inline Function** on the original function
5. Rename the new function

## Example

### Before

```java
public class HeatingController {
    public void adjustTemperature() {
        int currentTemp = thermostat.getCurrentTemperature();  // External query
        int targetTemp = thermostat.getTargetTemperature();    // External query

        if (currentTemp < targetTemp) {
            heater.turnOn();
        } else if (currentTemp > targetTemp + 2) {
            heater.turnOff();
        }
    }
}
```

### After

```java
public class HeatingController {
    public void adjustTemperature(int currentTemp, int targetTemp) {
        if (currentTemp < targetTemp) {
            heater.turnOn();
        } else if (currentTemp > targetTemp + 2) {
            heater.turnOff();
        }
    }
}

// Caller now responsible for the query
controller.adjustTemperature(
    thermostat.getCurrentTemperature(),
    thermostat.getTargetTemperature()
);
```

### Breaking Configuration Dependency

```java
// Before - hard to test
public class PricingService {
    @Inject
    Config config;

    public BigDecimal calculatePrice(Product product) {
        BigDecimal basePrice = product.getBasePrice();
        BigDecimal taxRate = config.getTaxRate();  // Internal query
        return basePrice.multiply(BigDecimal.ONE.add(taxRate));
    }
}

// After - easier to test
public class PricingService {
    public BigDecimal calculatePrice(Product product, BigDecimal taxRate) {
        BigDecimal basePrice = product.getBasePrice();
        return basePrice.multiply(BigDecimal.ONE.add(taxRate));
    }
}

// Test becomes simple
@Test
void calculatesWithTax() {
    BigDecimal price = pricingService.calculatePrice(product, new BigDecimal("0.20"));
    assertEquals(new BigDecimal("120.00"), price);
}
```

### Moving Responsibility Up

```java
// Before - tightly coupled to request context
public class UserService {
    public User getCurrentUser() {
        String userId = SecurityContext.getCurrentUserId();  // Hidden dependency
        return userRepository.findById(userId);
    }
}

// After - explicit dependency
public class UserService {
    public User getUser(String userId) {
        return userRepository.findById(userId);
    }
}

// Caller decides where to get userId
User user = userService.getUser(request.getUserId());
```

## Trade-offs

This refactoring moves responsibility to callers:

- **Pro**: More testable, explicit dependencies, no hidden state
- **Pro**: Function becomes more reusable and pure
- **Con**: Callers become more complex
- **Con**: May create duplication if many callers need same query

## When to Use

✅ **Use Replace Query with Parameter when:**

- You want to remove a hidden dependency
- Testing requires mocking the query source
- You want the function to be more pure
- Different callers need different query sources

❌ **Avoid Replace Query with Parameter when:**

- The query is stable and always the same
- Adding the parameter would burden all callers
- The query is an implementation detail callers shouldn't know
- It would push too much complexity to callers

## Related Refactorings

- **Replace Parameter with Query**: The inverse refactoring
- **Extract Function**: Often used in the mechanics
- **Introduce Parameter Object**: When adding multiple parameters
