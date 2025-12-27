# Remove Flag Argument

_Also known as: Replace Parameter with Explicit Methods_

## Intent

Replace a boolean or enum flag argument with separate functions for each case.

## Code Smells That Indicate This Refactoring

- Boolean parameter that changes function behavior
- Callers always pass literal true/false
- Function does very different things based on flag
- Flag makes the call site hard to understand

## Mechanics

1. Create an explicit function for each value of the flag parameter
2. For each caller, determine which case they want and use the appropriate new function
3. Test after each caller change
4. When all callers use the new functions, remove the original function

## Example

### Before

```java
public class BookingService {
    public Booking createBooking(Customer customer, boolean isPriority) {
        if (isPriority) {
            // priority booking logic
            return new PriorityBooking(customer, LocalDateTime.now(), generatePriorityCode());
        } else {
            // regular booking logic
            return new RegularBooking(customer, LocalDateTime.now());
        }
    }
}

// Usage - what does 'true' mean?
Booking booking = service.createBooking(customer, true);
Booking regular = service.createBooking(customer, false);
```

### After

```java
public class BookingService {
    public Booking createPriorityBooking(Customer customer) {
        return new PriorityBooking(customer, LocalDateTime.now(), generatePriorityCode());
    }

    public Booking createRegularBooking(Customer customer) {
        return new RegularBooking(customer, LocalDateTime.now());
    }
}

// Usage - clear intent
Booking booking = service.createPriorityBooking(customer);
Booking regular = service.createRegularBooking(customer);
```

### Enum Flag Example

```java
// Before
public class ReportGenerator {
    public Report generate(ReportType type) {
        switch (type) {
            case SUMMARY:
                return generateSummaryReport();
            case DETAILED:
                return generateDetailedReport();
            case EXECUTIVE:
                return generateExecutiveReport();
            default:
                throw new IllegalArgumentException();
        }
    }
}

// After
public class ReportGenerator {
    public Report generateSummaryReport() {
        // summary logic
    }

    public Report generateDetailedReport() {
        // detailed logic
    }

    public Report generateExecutiveReport() {
        // executive logic
    }
}
```

### Keeping Internal Method

```java
public class ShippingCalculator {
    // Public API - clear intent
    public BigDecimal calculateStandardShipping(Order order) {
        return calculateShipping(order, ShippingMethod.STANDARD);
    }

    public BigDecimal calculateExpressShipping(Order order) {
        return calculateShipping(order, ShippingMethod.EXPRESS);
    }

    // Private implementation - flag is fine internally
    private BigDecimal calculateShipping(Order order, ShippingMethod method) {
        BigDecimal baseRate = order.getWeight().multiply(method.getRatePerKg());
        if (method == ShippingMethod.EXPRESS) {
            baseRate = baseRate.add(method.getSurcharge());
        }
        return baseRate;
    }
}
```

## When NOT a Flag Argument

The refactoring is less necessary when the argument comes from data:

```java
// This is fine - the flag comes from data, not a literal
public void setStatus(boolean isActive) {
    // ...
}

// Called with actual variable, not literal
user.setStatus(preferences.isActive());
```

## When to Use

✅ **Use Remove Flag Argument when:**

- Callers always pass literal values (true/false/enum constant)
- The flag significantly changes behavior
- Call sites are hard to understand
- The flag leads to complex if-else in the function

❌ **Avoid Remove Flag Argument when:**

- The flag value comes from data, not literals
- There are many possible values
- Behavior difference is minor
- Creating multiple functions would cause duplication

## Related Refactorings

- **Parameterize Function**: The inverse refactoring
- **Decompose Conditional**: Often applied inside the function
- **Replace Conditional with Polymorphism**: Alternative approach
