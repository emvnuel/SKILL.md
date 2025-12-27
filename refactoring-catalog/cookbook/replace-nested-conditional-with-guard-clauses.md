# Replace Nested Conditional with Guard Clauses

## Intent

Use guard clauses for special cases that return early, leaving the main logic un-nested.

## Code Smells That Indicate This Refactoring

- Deeply nested if-else structures
- Main logic buried inside multiple conditions
- Special cases handled with else branches
- Arrow-shaped code (increasing indentation)

## Mechanics

1. Select the outermost condition to turn into a guard clause
2. If the condition is a guard, replace with a guard clause that returns early
3. Test
4. Repeat for remaining conditions

## Example

### Before

```java
public class PaymentCalculator {
    public double calculatePayment(Employee employee) {
        double result;
        if (employee.isSeparated()) {
            result = calculateSeparatedAmount(employee);
        } else {
            if (employee.isRetired()) {
                result = calculateRetiredAmount(employee);
            } else {
                result = calculateNormalPayment(employee);
            }
        }
        return result;
    }
}
```

### After

```java
public class PaymentCalculator {
    public double calculatePayment(Employee employee) {
        if (employee.isSeparated()) {
            return calculateSeparatedAmount(employee);
        }
        if (employee.isRetired()) {
            return calculateRetiredAmount(employee);
        }
        return calculateNormalPayment(employee);
    }
}
```

### Reversing Conditions

```java
// Before
public double getAdjustedCapital(Investment investment) {
    double result = 0;
    if (investment.getCapital() > 0) {
        if (investment.getInterestRate() > 0 && investment.getDuration() > 0) {
            result = (investment.getIncome() / investment.getDuration())
                   * investment.getAdjustmentFactor();
        }
    }
    return result;
}

// After
public double getAdjustedCapital(Investment investment) {
    if (investment.getCapital() <= 0) {
        return 0;
    }
    if (investment.getInterestRate() <= 0 || investment.getDuration() <= 0) {
        return 0;
    }
    return (investment.getIncome() / investment.getDuration())
         * investment.getAdjustmentFactor();
}
```

### Validation Guards

```java
public class OrderProcessor {
    public OrderResult processOrder(Order order) {
        // Guard clauses for validation
        if (order == null) {
            throw new IllegalArgumentException("Order cannot be null");
        }
        if (order.getItems().isEmpty()) {
            return OrderResult.failure("Order has no items");
        }
        if (!order.getCustomer().isActive()) {
            return OrderResult.failure("Customer account is not active");
        }
        if (!order.hasValidPayment()) {
            return OrderResult.failure("Invalid payment method");
        }

        // Main logic, un-nested
        processPayment(order);
        reserveInventory(order);
        scheduleShipping(order);
        return OrderResult.success(order);
    }
}
```

## When to Use

✅ **Use Replace Nested Conditional with Guard Clauses when:**

- You have deeply nested conditionals
- Special cases clutter the main logic
- Early returns would simplify the code
- You want to reduce indentation

❌ **Avoid Replace Nested Conditional with Guard Clauses when:**

- Multiple returns would make code harder to follow
- Team conventions require single return point
- The nested structure better represents the logic
- You need to ensure cleanup code runs (use try-finally instead)

## Related Refactorings

- **Decompose Conditional**: Extract complex conditions
- **Consolidate Conditional Expression**: Combine related guards
- **Replace Conditional with Polymorphism**: For type-based branching
