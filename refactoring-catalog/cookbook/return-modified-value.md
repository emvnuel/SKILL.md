# Return Modified Value

## Intent

Have a function return the modified value rather than updating data in place, making data flow clearer.

## Code Smells That Indicate This Refactoring

- Function modifies data passed to it via side effects
- Unclear what data is changed by a function call
- Hard to track data flow through the code
- Parallel processing is difficult due to mutations

## Mechanics

1. If the variable isn't already within the function, apply **Slide Statements** to bring initialization inside
2. Change the function to return the modified value
3. At each call site, replace the variable assignment with the function return
4. Test

## Example

### Before

```java
public class GpsCalculator {
    public void calculateDistance(TotalDistance totalDistance, Point point) {
        totalDistance.addDistance(calculateDistanceFromStart(point));
    }
}

public class TotalDistance {
    private double total = 0;

    public void addDistance(double distance) {
        this.total += distance;
    }

    public double getTotal() {
        return total;
    }
}

// Usage
TotalDistance totalDistance = new TotalDistance();
for (Point point : points) {
    calculator.calculateDistance(totalDistance, point);
}
return totalDistance.getTotal();
```

### After

```java
public class GpsCalculator {
    public double calculateDistance(double totalDistance, Point point) {
        return totalDistance + calculateDistanceFromStart(point);
    }
}

// Usage
double totalDistance = 0;
for (Point point : points) {
    totalDistance = calculator.calculateDistance(totalDistance, point);
}
return totalDistance;
```

### Collection Example

```java
// Before - mutates the list
public void addValidItems(List<Item> items, List<Item> validItems) {
    for (Item item : items) {
        if (item.isValid()) {
            validItems.add(item);
        }
    }
}

List<Item> validItems = new ArrayList<>();
addValidItems(items, validItems);

// After - returns the result
public List<Item> filterValidItems(List<Item> items) {
    List<Item> validItems = new ArrayList<>();
    for (Item item : items) {
        if (item.isValid()) {
            validItems.add(item);
        }
    }
    return validItems;
}

List<Item> validItems = filterValidItems(items);

// Even better with streams
public List<Item> filterValidItems(List<Item> items) {
    return items.stream()
        .filter(Item::isValid)
        .collect(Collectors.toList());
}
```

### Accumulator Pattern

```java
// Before - updates accumulator via side effect
public class OrderProcessor {
    public void addTax(OrderTotals totals) {
        double tax = totals.getSubtotal() * 0.1;
        totals.setTax(tax);
    }

    public void addShipping(OrderTotals totals) {
        double shipping = totals.getWeight() > 10 ? 15.0 : 5.0;
        totals.setShipping(shipping);
    }
}

// After - each step returns new value
public class OrderProcessor {
    public OrderTotals withTax(OrderTotals totals) {
        double tax = totals.getSubtotal() * 0.1;
        return totals.withTax(tax);
    }

    public OrderTotals withShipping(OrderTotals totals) {
        double shipping = totals.getWeight() > 10 ? 15.0 : 5.0;
        return totals.withShipping(shipping);
    }
}

// Fluent usage
OrderTotals result = processor.withTax(
    processor.withShipping(totals)
);
```

## When to Use

✅ **Use Return Modified Value when:**

- You want to make data flow explicit
- Side effects make code hard to understand
- You want to enable functional style
- Testing is easier with return values

❌ **Avoid Return Modified Value when:**

- The object is large and copying is expensive
- Mutation is more natural for the domain
- Multiple values need to be modified together
- Performance constraints require in-place mutation

## Related Refactorings

- **Separate Query from Modifier**: Related to clarifying side effects
- **Replace Temp with Query**: May enable returning values
- **Slide Statements**: Often used in mechanics
