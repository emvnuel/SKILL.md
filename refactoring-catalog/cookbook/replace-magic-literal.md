# Replace Magic Literal

_Also known as: Replace Magic Number with Symbolic Constant_

## Intent

Replace a literal value with a named constant to make its purpose clear and enable easy updates.

## Code Smells That Indicate This Refactoring

- Literal values whose meaning isn't obvious
- Same literal used in multiple places
- Value that might need to change in the future
- Numeric value representing a domain concept

## Mechanics

1. Create a constant and set it to the magic literal
2. Find all uses of the magic literal
3. Replace each with the constant
4. Test

## Example

### Before

```java
public class Employee {
    public double calculatePotentialEnergy(double mass, double height) {
        return mass * 9.81 * height;
    }

    public double getBaseSalary() {
        return 2500.0 * 12;
    }

    public boolean isEligibleForBonus() {
        return this.yearsOfService > 5;
    }

    public double calculateDiscount(double amount) {
        if (amount > 1000) {
            return amount * 0.05;
        }
        return 0;
    }
}
```

### After

```java
public class Employee {
    private static final double GRAVITATIONAL_ACCELERATION = 9.81; // m/s²
    private static final double MONTHLY_BASE_SALARY = 2500.0;
    private static final int MONTHS_PER_YEAR = 12;
    private static final int BONUS_ELIGIBILITY_YEARS = 5;
    private static final double LARGE_ORDER_THRESHOLD = 1000.0;
    private static final double LARGE_ORDER_DISCOUNT_RATE = 0.05;

    public double calculatePotentialEnergy(double mass, double height) {
        return mass * GRAVITATIONAL_ACCELERATION * height;
    }

    public double getAnnualBaseSalary() {
        return MONTHLY_BASE_SALARY * MONTHS_PER_YEAR;
    }

    public boolean isEligibleForBonus() {
        return this.yearsOfService > BONUS_ELIGIBILITY_YEARS;
    }

    public double calculateDiscount(double amount) {
        if (amount > LARGE_ORDER_THRESHOLD) {
            return amount * LARGE_ORDER_DISCOUNT_RATE;
        }
        return 0;
    }
}
```

### Using Enums for Related Constants

```java
public enum HttpStatus {
    OK(200),
    CREATED(201),
    BAD_REQUEST(400),
    NOT_FOUND(404),
    INTERNAL_ERROR(500);

    private final int code;

    HttpStatus(int code) {
        this.code = code;
    }

    public int getCode() {
        return code;
    }
}

// Usage
if (response.getStatusCode() == HttpStatus.NOT_FOUND.getCode()) {
    handleNotFound();
}
```

### Using Configuration for Variable Values

```java
@ApplicationScoped
public class PricingService {
    @Inject
    @ConfigProperty(name = "pricing.discount.threshold", defaultValue = "1000.0")
    double discountThreshold;

    @Inject
    @ConfigProperty(name = "pricing.discount.rate", defaultValue = "0.05")
    double discountRate;

    public double calculateDiscount(double amount) {
        if (amount > discountThreshold) {
            return amount * discountRate;
        }
        return 0;
    }
}
```

## When to Use

✅ **Use Replace Magic Literal when:**

- The literal's meaning isn't immediately obvious
- The same literal appears in multiple places
- The value might change in the future
- You want self-documenting code

❌ **Avoid Replace Magic Literal when:**

- The meaning is genuinely obvious (0 for empty, 1 for single)
- The literal is already in a well-named method
- Creating a constant would be more confusing than the literal
- The value is used only once in an obvious context

## Related Refactorings

- **Replace Primitive with Object**: When the literal needs more behavior
- **Extract Variable**: For complex expressions containing literals
- **Introduce Parameter Object**: For grouping related constants
