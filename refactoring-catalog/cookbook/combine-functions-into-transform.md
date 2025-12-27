# Combine Functions into Transform

## Intent

Combine related functions that calculate derived data into a transform that produces an enriched version of the source data.

## Code Smells That Indicate This Refactoring

- Multiple functions derive data from the same source
- Derived data is calculated repeatedly
- Related calculations should be done together
- Need to pass calculated values between functions

## Mechanics

1. Create a transform function that takes the source data and returns a copy
2. Move the calculation logic into the transform function
3. Add each calculated field to the returned object
4. At call sites, use the enriched object instead of calculating

## Example

### Before

```java
public class OrderAnalyzer {
    public double getBaseCharge(Order order) {
        return order.getQuantity() * order.getItemPrice();
    }

    public double getTaxableCharge(Order order) {
        return Math.max(0, getBaseCharge(order) - getTaxThreshold());
    }

    public double getTax(Order order) {
        return getTaxableCharge(order) * TAX_RATE;
    }

    public double getTotal(Order order) {
        return getBaseCharge(order) + getTax(order);
    }
}

// Usage - calculations happen multiple times
Order order = getOrder();
double base = analyzer.getBaseCharge(order);
double tax = analyzer.getTax(order);
double total = analyzer.getTotal(order);
```

### After

```java
public record EnrichedOrder(
    Order original,
    double baseCharge,
    double taxableCharge,
    double tax,
    double total
) {}

public class OrderEnricher {
    public EnrichedOrder enrich(Order order) {
        double baseCharge = order.getQuantity() * order.getItemPrice();
        double taxableCharge = Math.max(0, baseCharge - getTaxThreshold());
        double tax = taxableCharge * TAX_RATE;
        double total = baseCharge + tax;

        return new EnrichedOrder(order, baseCharge, taxableCharge, tax, total);
    }
}

// Usage - calculations done once
Order order = getOrder();
EnrichedOrder enriched = enricher.enrich(order);
double base = enriched.baseCharge();
double tax = enriched.tax();
double total = enriched.total();
```

### Using Builder Pattern

```java
public class OrderSummary {
    private final Order order;
    private final double baseCharge;
    private final double discount;
    private final double shipping;
    private final double tax;
    private final double total;

    private OrderSummary(Builder builder) {
        this.order = builder.order;
        this.baseCharge = builder.baseCharge;
        this.discount = builder.discount;
        this.shipping = builder.shipping;
        this.tax = builder.tax;
        this.total = builder.total;
    }

    public static OrderSummary create(Order order, Customer customer, Address address) {
        double baseCharge = calculateBaseCharge(order);
        double discount = calculateDiscount(order, customer);
        double shipping = calculateShipping(order, address);
        double taxableAmount = baseCharge - discount;
        double tax = calculateTax(taxableAmount, address);
        double total = taxableAmount + shipping + tax;

        return new Builder(order)
            .baseCharge(baseCharge)
            .discount(discount)
            .shipping(shipping)
            .tax(tax)
            .total(total)
            .build();
    }

    // Getters and Builder class...
}
```

## Transform vs Class

| Combine into Class             | Combine into Transform     |
| ------------------------------ | -------------------------- |
| Original data is mutable       | Original data is immutable |
| Update logic can modify source | Source is never modified   |
| Natural OO encapsulation       | Functional/pipeline style  |
| Methods on the object          | Enriched copy of data      |

## When to Use

✅ **Use Combine Functions into Transform when:**

- Working with immutable data
- You want to enrich data for later use
- Multiple consumers need the derived values
- Functional style is preferred

❌ **Avoid Combine Functions into Transform when:**

- Source data is mutable (use class instead)
- Transformation is expensive and rarely needed
- A class would be more natural
- Derived values change over time

## Related Refactorings

- **Combine Functions into Class**: Alternative approach
- **Extract Variable**: For intermediate calculations
- **Introduce Parameter Object**: May be combined with
