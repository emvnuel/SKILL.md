# Inline Function

_Also known as: Inline Method_

## Intent

Replace a function call with the body of the function when the function body is as clear as the name.

## Code Smells That Indicate This Refactoring

- Function body is as simple as its name
- Too many levels of indirection
- Function does nothing but delegate to another
- Group of functions that are poorly organized

## Mechanics

1. Check that the function isn't polymorphic (not overridden in subclasses)
2. Find all callers of the function
3. Replace each call with the function's body
4. Test after each replacement
5. Remove the function definition

## Example

### Before

```java
public class RatingCalculator {
    public int getRating(Driver driver) {
        return moreThanFiveLateDeliveries(driver) ? 2 : 1;
    }

    private boolean moreThanFiveLateDeliveries(Driver driver) {
        return driver.getNumberOfLateDeliveries() > 5;
    }
}
```

### After

```java
public class RatingCalculator {
    public int getRating(Driver driver) {
        return driver.getNumberOfLateDeliveries() > 5 ? 2 : 1;
    }
}
```

### Inlining Delegating Methods

```java
// Before
public class ReportService {
    public List<String> reportLines(Customer customer) {
        List<String> lines = new ArrayList<>();
        gatherCustomerData(lines, customer);
        return lines;
    }

    private void gatherCustomerData(List<String> lines, Customer customer) {
        lines.add("name: " + customer.getName());
        lines.add("location: " + customer.getLocation());
    }
}

// After
public class ReportService {
    public List<String> reportLines(Customer customer) {
        List<String> lines = new ArrayList<>();
        lines.add("name: " + customer.getName());
        lines.add("location: " + customer.getLocation());
        return lines;
    }
}
```

### Simplifying Before Reextraction

```java
// Before - messy structure
public double baseCharge(int usage) {
    return chargeFor(usage);
}

private double chargeFor(int usage) {
    return usage * getRate();
}

private double getRate() {
    return 0.05;
}

// Inline everything first
public double baseCharge(int usage) {
    return usage * 0.05;
}

// Then re-extract with better structure
public double baseCharge(int usage) {
    return usage * getBaseRate();
}

private static final double BASE_RATE = 0.05;

private double getBaseRate() {
    return BASE_RATE;
}
```

## When to Use

✅ **Use Inline Function when:**

- The function body is as clear as the name
- You have a group of functions that are badly factored
- The function is just one line delegating to another
- Before re-extracting with better boundaries

❌ **Avoid Inline Function when:**

- The function is polymorphic (overridden in subclasses)
- The function is complex and the name provides value
- The function is called from many places
- Inlining would create duplication

## Related Refactorings

- **Extract Function**: The inverse operation
- **Inline Variable**: Similar for variables
- **Move Function**: May be done instead
