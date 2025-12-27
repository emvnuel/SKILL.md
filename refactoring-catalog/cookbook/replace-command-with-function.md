# Replace Command with Function

## Intent

Replace a command object with a simple function when the command is no longer complex enough to justify the extra structure.

## Code Smells That Indicate This Refactoring

- Command class does one simple thing
- No undo/redo or queuing needed
- Command has become over-engineered
- The indirection makes code harder to understand

## Mechanics

1. Create a function that does what the command's execute method does
2. Replace each use of the command with a function call
3. Test after each replacement
4. Delete the command class

## Example

### Before

```java
public class ChargeCalculatorCommand {
    private final Customer customer;
    private final Usage usage;
    private final Provider provider;

    public ChargeCalculatorCommand(Customer customer, Usage usage, Provider provider) {
        this.customer = customer;
        this.usage = usage;
        this.provider = provider;
    }

    public BigDecimal execute() {
        BigDecimal baseCharge = customer.getBaseRate().multiply(usage.getAmount());
        return baseCharge.add(provider.getConnectionCharge());
    }
}

// Usage
BigDecimal charge = new ChargeCalculatorCommand(customer, usage, provider).execute();
```

### After

```java
public class ChargeCalculator {
    public static BigDecimal calculate(Customer customer, Usage usage, Provider provider) {
        BigDecimal baseCharge = customer.getBaseRate().multiply(usage.getAmount());
        return baseCharge.add(provider.getConnectionCharge());
    }
}

// Usage
BigDecimal charge = ChargeCalculator.calculate(customer, usage, provider);
```

### As Instance Method

```java
// If the function naturally belongs to a class
public class Customer {
    public BigDecimal calculateCharge(Usage usage, Provider provider) {
        BigDecimal baseCharge = getBaseRate().multiply(usage.getAmount());
        return baseCharge.add(provider.getConnectionCharge());
    }
}

// Usage
BigDecimal charge = customer.calculateCharge(usage, provider);
```

### Simplifying Over-Engineered Code

```java
// Before - over-engineered for a simple operation
public interface Validator {
    ValidationResult validate(Object input);
}

public class EmailValidator implements Validator {
    @Override
    public ValidationResult validate(Object input) {
        String email = (String) input;
        if (email == null || !email.contains("@")) {
            return ValidationResult.invalid("Invalid email");
        }
        return ValidationResult.valid();
    }
}

// After - simple function is sufficient
public class Validators {
    public static ValidationResult validateEmail(String email) {
        if (email == null || !email.contains("@")) {
            return ValidationResult.invalid("Invalid email");
        }
        return ValidationResult.valid();
    }
}
```

## When to Use

✅ **Use Replace Command with Function when:**

- The command does one simple thing
- Command features (undo, queue) are not used
- The class just wraps a single method
- Function call would be clearer

❌ **Avoid Replace Command with Function when:**

- Undo/redo functionality is needed
- Commands are queued or scheduled
- Multiple related commands share structure
- The command pattern adds value

## Related Refactorings

- **Replace Function with Command**: The inverse
- **Inline Class**: Similar simplification
- **Inline Function**: When the function itself becomes trivial
