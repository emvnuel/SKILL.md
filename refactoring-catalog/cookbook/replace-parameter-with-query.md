# Replace Parameter with Query

_Also known as: Replace Parameter with Method_

## Intent

Remove a parameter that can be derived from another parameter or from the function's context.

## Code Smells That Indicate This Refactoring

- Parameter can be calculated from other parameters
- Caller always computes the value the same way
- Parameter is redundant given the function's context
- Parameter makes the API harder to use

## Mechanics

1. If necessary, extract the calculation of the parameter into a function
2. Remove the parameter from the function
3. Replace references to the parameter with calls to the derived calculation
4. Test

## Example

### Before

```java
public class Order {
    private int quantity;
    private double itemPrice;

    public double getDiscountedPrice(int basePrice, int discountLevel) {
        switch (discountLevel) {
            case 1: return basePrice * 0.95;
            case 2: return basePrice * 0.90;
            default: return basePrice;
        }
    }

    public double getFinalPrice() {
        int basePrice = quantity * (int) itemPrice;
        return getDiscountedPrice(basePrice, getDiscountLevel());
    }

    private int getDiscountLevel() {
        return quantity > 100 ? 2 : 1;
    }
}
```

### After

```java
public class Order {
    private int quantity;
    private double itemPrice;

    public double getDiscountedPrice() {
        int basePrice = getBasePrice();
        switch (getDiscountLevel()) {
            case 1: return basePrice * 0.95;
            case 2: return basePrice * 0.90;
            default: return basePrice;
        }
    }

    public double getFinalPrice() {
        return getDiscountedPrice();
    }

    private int getBasePrice() {
        return quantity * (int) itemPrice;
    }

    private int getDiscountLevel() {
        return quantity > 100 ? 2 : 1;
    }
}
```

### Configuration Example

```java
// Before
public class ReportService {
    @Inject
    Config config;

    public Report generate(String templatePath, int maxPages) {
        // Uses templatePath and maxPages
    }
}

// Called as:
report.generate(config.getTemplatePath(), config.getMaxPages());

// After
public class ReportService {
    @Inject
    Config config;

    public Report generate() {
        String templatePath = config.getTemplatePath();
        int maxPages = config.getMaxPages();
        // Uses templatePath and maxPages
    }
}

// Called as:
report.generate();
```

## When NOT to Apply

Be careful about removing parameters in cases like this:

```java
// Keep the parameter - different callers may want different values
public class PricingService {
    public double applyDiscount(double price, double discountRate) {
        return price * (1 - discountRate);
    }
}

// Don't change to this - loses flexibility
public double applyDiscount(double price) {
    return price * (1 - config.getDefaultDiscountRate());
}
```

## When to Use

✅ **Use Replace Parameter with Query when:**

- The parameter is always calculated the same way
- The function has access to the data needed for calculation
- Removing the parameter simplifies the API
- The parameter adds no flexibility to the function

❌ **Avoid Replace Parameter with Query when:**

- Different callers need different values
- The calculation is expensive and can be cached by caller
- Removing the parameter creates unwanted coupling
- The query has side effects or is non-deterministic

## Related Refactorings

- **Replace Query with Parameter**: The inverse refactoring
- **Preserve Whole Object**: Alternative when passing object is better
- **Change Function Declaration**: Used to remove the parameter
