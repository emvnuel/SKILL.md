# Introduce Assertion

## Intent

Make a hidden assumption explicit by using an assertion.

## Code Smells That Indicate This Refactoring

- Code assumes a certain condition is always true
- Defensive programming with if statements that "should never happen"
- Comments documenting what should be true at a point in code
- Debugging difficult because invariants aren't explicit

## Mechanics

1. Identify a condition that is assumed to be true
2. Add an assertion for that condition
3. Test

## Example

### Before

```java
public class Employee {
    private double expenseLimit = -1; // -1 means no limit set, use project default
    private Project primaryProject;

    public double getExpenseLimit() {
        // Assumes project is set when limit is -1
        return expenseLimit != -1 ? expenseLimit : primaryProject.getMemberExpenseLimit();
    }
}
```

### After

```java
public class Employee {
    private double expenseLimit = -1;
    private Project primaryProject;

    public double getExpenseLimit() {
        assert expenseLimit != -1 || primaryProject != null
            : "Employee must have expense limit or assigned project";
        return expenseLimit != -1 ? expenseLimit : primaryProject.getMemberExpenseLimit();
    }
}
```

### Using Objects.requireNonNull

```java
public class OrderProcessor {
    public OrderResult process(Order order) {
        Objects.requireNonNull(order, "Order cannot be null");
        Objects.requireNonNull(order.getCustomer(), "Order must have a customer");

        // Now we can safely use order and customer
        return processValidOrder(order);
    }
}
```

### Precondition Checks

```java
public class Account {
    private BigDecimal balance;

    public void withdraw(BigDecimal amount) {
        // Precondition assertions
        assert amount.compareTo(BigDecimal.ZERO) > 0 : "Withdrawal amount must be positive";
        assert balance.compareTo(amount) >= 0 : "Insufficient funds";

        balance = balance.subtract(amount);
    }
}
```

### Using Guava Preconditions

```java
import static com.google.common.base.Preconditions.*;

public class ShippingCalculator {
    public BigDecimal calculate(Package pkg, Address destination) {
        checkNotNull(pkg, "Package cannot be null");
        checkNotNull(destination, "Destination cannot be null");
        checkArgument(pkg.getWeight() > 0, "Package weight must be positive");
        checkState(isInitialized(), "Calculator not initialized");

        return doCalculation(pkg, destination);
    }
}
```

### Class Invariant Assertions

```java
public class DateRange {
    private final LocalDate start;
    private final LocalDate end;

    public DateRange(LocalDate start, LocalDate end) {
        assert start != null : "Start date cannot be null";
        assert end != null : "End date cannot be null";
        assert !start.isAfter(end) : "Start must not be after end";

        this.start = start;
        this.end = end;
    }

    public DateRange shifted(long days) {
        DateRange result = new DateRange(
            start.plusDays(days),
            end.plusDays(days)
        );
        assertInvariant();
        return result;
    }

    private void assertInvariant() {
        assert !start.isAfter(end) : "Invariant violated: start after end";
    }
}
```

## When to Use

✅ **Use Introduce Assertion when:**

- Code assumes a condition should always be true
- You want to document programmer expectations
- Debugging is difficult without explicit invariants
- You want to fail fast during development

❌ **Avoid Introduce Assertion when:**

- The condition might legitimately be false
- You need production error handling (use exceptions)
- The assertion would slow down critical code paths
- Input validation is needed (use proper validation)

## Assertions vs Exceptions

| Use Assertions For      | Use Exceptions For            |
| ----------------------- | ----------------------------- |
| Programmer errors       | User/input errors             |
| Internal invariants     | External interface violations |
| Development-time checks | Runtime error handling        |
| Should never happen     | Can legitimately happen       |

## Related Refactorings

- **Replace Exception with Precheck**: Related validation technique
- **Introduce Special Case**: For handling expected special values
- **Extract Function**: To extract validation logic
