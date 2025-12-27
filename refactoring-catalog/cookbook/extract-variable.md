# Extract Variable

_Also known as: Introduce Explaining Variable_

## Intent

Extract a complex expression into a named variable that explains its purpose.

## Code Smells That Indicate This Refactoring

- Complex expression that's hard to understand
- Same expression calculated multiple times
- Expression needs a comment to explain what it means
- Long conditional with multiple parts

## Mechanics

1. Ensure the expression has no side effects
2. Declare an immutable variable and set it to a copy of the expression
3. Replace the original expression with the new variable
4. Test

## Example

### Before

```java
public class PriceCalculator {
    public double price(Order order) {
        // price is base price - quantity discount + shipping
        return order.getQuantity() * order.getItemPrice() -
            Math.max(0, order.getQuantity() - 500) * order.getItemPrice() * 0.05 +
            Math.min(order.getQuantity() * order.getItemPrice() * 0.1, 100);
    }
}
```

### After

```java
public class PriceCalculator {
    public double price(Order order) {
        final double basePrice = order.getQuantity() * order.getItemPrice();
        final double quantityDiscount = Math.max(0, order.getQuantity() - 500) * order.getItemPrice() * 0.05;
        final double shipping = Math.min(basePrice * 0.1, 100);
        return basePrice - quantityDiscount + shipping;
    }
}
```

### Extracting in Conditionals

```java
// Before
public double getPayAmount(Employee employee) {
    if (employee.isSeparated() ||
        employee.getYearsEmployed() < 2 ||
        employee.isRetired()) {
        return 0;
    }
    return employee.calculatePay();
}

// After
public double getPayAmount(Employee employee) {
    final boolean isNotEligible = employee.isSeparated() ||
                                  employee.getYearsEmployed() < 2 ||
                                  employee.isRetired();
    if (isNotEligible) {
        return 0;
    }
    return employee.calculatePay();
}
```

### Class Context - Consider Method Instead

```java
// If the expression is needed in multiple places in a class,
// consider extracting as a method instead

public class Order {
    private int quantity;
    private double itemPrice;

    public double getBasePrice() {
        return quantity * itemPrice;
    }

    public double getQuantityDiscount() {
        return Math.max(0, quantity - 500) * itemPrice * 0.05;
    }

    public double getShipping() {
        return Math.min(getBasePrice() * 0.1, 100);
    }

    public double getPrice() {
        return getBasePrice() - getQuantityDiscount() + getShipping();
    }
}
```

## When to Use

✅ **Use Extract Variable when:**

- Expression is complex and needs explanation
- Expression is used multiple times in the same scope
- You're working within a single function
- Breaking down a complex expression step by step

❌ **Avoid Extract Variable when:**

- The expression is already clear
- The variable would only be used once and adds no clarity
- The expression should be shared across the class (use method)
- Adding a name adds no value

## Related Refactorings

- **Inline Variable**: The inverse operation
- **Replace Temp with Query**: When the variable should be a method
- **Extract Function**: For larger extractions
