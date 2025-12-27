# Inline Variable

_Also known as: Inline Temp_

## Intent

Remove a variable that adds no value and replace references with the expression directly.

## Code Smells That Indicate This Refactoring

- Variable adds no meaning to the expression
- Variable is only used once
- Variable name doesn't explain more than the expression
- Variable gets in the way of other refactorings

## Mechanics

1. Check that the right-hand side of the assignment has no side effects
2. Make the variable final if it isn't already (to ensure it's only assigned once)
3. Find the first reference to the variable and replace it with the expression
4. Test
5. Repeat for each reference
6. Remove the declaration and assignment

## Example

### Before

```java
public boolean isExpensive(Order order) {
    double basePrice = order.getBasePrice();
    return basePrice > 1000;
}
```

### After

```java
public boolean isExpensive(Order order) {
    return order.getBasePrice() > 1000;
}
```

### Preparation for Further Refactoring

```java
// Before - variable blocks Extract Function
public void process(Order order) {
    double basePrice = order.getBasePrice();
    if (basePrice > 1000) {
        // ... do something with expensive order
    }
    double discount = basePrice * 0.1;
    // ... more code
}

// After inlining - now can extract more easily
public void process(Order order) {
    if (order.getBasePrice() > 1000) {
        processExpensiveOrder(order);
    }
    double discount = order.getBasePrice() * 0.1;
    // ... more code
}
```

### When the Variable Adds Value - Don't Inline

```java
// DON'T inline this - the name adds clarity
double millimeters = inches * 25.4;
return millimeters;

// DON'T inline this - used multiple times
double basePrice = order.getBasePrice();
if (basePrice > 1000) {
    applyExpensiveDiscount(basePrice);
}
return basePrice;
```

## When to Use

✅ **Use Inline Variable when:**

- The variable name doesn't communicate more than the expression
- The variable is getting in the way of refactoring
- The variable is only used once
- The assignment is simple and has no side effects

❌ **Avoid Inline Variable when:**

- The variable name adds meaning
- The variable is used multiple times
- The expression is complex or long
- The variable helps debugging

## Related Refactorings

- **Extract Variable**: The inverse operation
- **Inline Function**: Similar for functions
- **Replace Temp with Query**: Alternative when expression should be method
