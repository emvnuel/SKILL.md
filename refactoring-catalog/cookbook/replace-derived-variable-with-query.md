# Replace Derived Variable with Query

## Intent

Replace a variable that stores a calculated value with a method that computes it on demand.

## Code Smells That Indicate This Refactoring

- Variable that's calculated from other values
- Stale data due to inconsistent updates
- Need to check if variable is up-to-date
- Mutable derived data that could be computed

## Mechanics

1. Identify all points where the variable is updated
2. Create a method that calculates the value
3. Apply **Introduce Assertion** to verify calculations match
4. Test
5. Replace reads of the variable with calls to the method
6. Test
7. Delete the variable declaration and updates

## Example

### Before

```java
public class ProductionPlan {
    private int production;
    private List<Adjustment> adjustments = new ArrayList<>();

    public int getProduction() {
        return production;
    }

    public void applyAdjustment(Adjustment adjustment) {
        adjustments.add(adjustment);
        production += adjustment.getAmount();
    }
}
```

### After

```java
public class ProductionPlan {
    private List<Adjustment> adjustments = new ArrayList<>();

    public int getProduction() {
        return adjustments.stream()
            .mapToInt(Adjustment::getAmount)
            .sum();
    }

    public void applyAdjustment(Adjustment adjustment) {
        adjustments.add(adjustment);
    }
}
```

### Shopping Cart Example

```java
// Before
public class ShoppingCart {
    private List<CartItem> items = new ArrayList<>();
    private BigDecimal total = BigDecimal.ZERO;

    public void addItem(CartItem item) {
        items.add(item);
        total = total.add(item.getPrice());  // Manual sync
    }

    public void removeItem(CartItem item) {
        items.remove(item);
        total = total.subtract(item.getPrice());  // Manual sync
    }

    public BigDecimal getTotal() {
        return total;  // Could be stale!
    }
}

// After
public class ShoppingCart {
    private List<CartItem> items = new ArrayList<>();

    public void addItem(CartItem item) {
        items.add(item);
    }

    public void removeItem(CartItem item) {
        items.remove(item);
    }

    public BigDecimal getTotal() {
        return items.stream()
            .map(CartItem::getPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

### With Caching

When the computation is expensive:

```java
public class ProductionPlan {
    private List<Adjustment> adjustments = new ArrayList<>();
    private Integer cachedProduction = null;  // Lazy cache

    public int getProduction() {
        if (cachedProduction == null) {
            cachedProduction = calculateProduction();
        }
        return cachedProduction;
    }

    public void applyAdjustment(Adjustment adjustment) {
        adjustments.add(adjustment);
        cachedProduction = null;  // Invalidate cache
    }

    private int calculateProduction() {
        return adjustments.stream()
            .mapToInt(Adjustment::getAmount)
            .sum();
    }
}
```

## When to Use

✅ **Use Replace Derived Variable with Query when:**

- Variable stores a value derived from other data
- Keeping the variable in sync is error-prone
- The calculation is simple
- You want to guarantee freshness

❌ **Avoid Replace Derived Variable with Query when:**

- Calculation is expensive and called frequently
- The source data is immutable (no sync issues)
- Performance is critical
- Variable is set independently, not derived

## Trade-offs

| Stored Variable                   | Query                          |
| --------------------------------- | ------------------------------ |
| Fast access                       | Always current                 |
| Can become stale                  | Calculated each time           |
| More code to maintain sync        | Simpler code                   |
| Better for expensive calculations | Better for simple calculations |

## Related Refactorings

- **Replace Temp with Query**: Similar for local variables
- **Encapsulate Variable**: May be needed first
- **Extract Function**: For the calculation logic
