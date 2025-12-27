# Replace Temp with Query

## Intent

Replace a temporary variable with a method call that computes the value.

## Code Smells That Indicate This Refactoring

- Temporary variable used to hold the result of an expression
- The same calculation needed elsewhere in the class
- Variable is used in multiple places within a method
- Preparing for extraction of methods

## Mechanics

1. Check that the variable is calculated once and not modified after
2. Extract the expression into a method that returns the value
3. Replace the variable reference with a call to the new method
4. Test

## Example

### Before

```java
public class Order {
    private int quantity;
    private double itemPrice;

    public double getPrice() {
        double basePrice = quantity * itemPrice;
        if (basePrice > 1000) {
            return basePrice * 0.95;
        } else {
            return basePrice * 0.98;
        }
    }
}
```

### After

```java
public class Order {
    private int quantity;
    private double itemPrice;

    public double getPrice() {
        if (getBasePrice() > 1000) {
            return getBasePrice() * 0.95;
        } else {
            return getBasePrice() * 0.98;
        }
    }

    private double getBasePrice() {
        return quantity * itemPrice;
    }
}
```

### With Multiple Variables

```java
// Before
public double calculateTotal() {
    double basePrice = quantity * itemPrice;
    double discountFactor = basePrice > 1000 ? 0.95 : 0.98;
    return basePrice * discountFactor;
}

// After
public double calculateTotal() {
    return getBasePrice() * getDiscountFactor();
}

private double getBasePrice() {
    return quantity * itemPrice;
}

private double getDiscountFactor() {
    return getBasePrice() > 1000 ? 0.95 : 0.98;
}
```

## Performance Considerations

If the calculation is expensive:

```java
// Option 1: Cache the result
private Double cachedBasePrice;

private double getBasePrice() {
    if (cachedBasePrice == null) {
        cachedBasePrice = quantity * itemPrice;
    }
    return cachedBasePrice;
}

// Option 2: Use local variable in hot paths
public void hotPath() {
    double basePrice = getBasePrice();  // Calculate once
    for (int i = 0; i < 1000; i++) {
        process(basePrice);  // Reuse
    }
}
```

## When to Use

✅ **Use Replace Temp with Query when:**

- The calculation is needed in multiple places
- You want to extract a method and the temp is in the way
- The expression defines a meaningful concept
- The value doesn't change after initial calculation

❌ **Avoid Replace Temp with Query when:**

- The calculation is expensive and called frequently
- The variable is assigned multiple times
- The temporary is genuinely temporary (loop variable)
- The method would have too many calls to other methods

## Related Refactorings

- **Extract Variable**: The inverse (may be needed first)
- **Extract Function**: Often follows this refactoring
- **Inline Variable**: Applied before extraction
- **Split Variable**: If the temp is assigned multiple times
