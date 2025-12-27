# Slide Statements

_Also known as: Consolidate Duplicate Conditional Fragments_

## Intent

Move statements closer to where they are used to make the code easier to understand and prepare for other refactorings.

## Code Smells That Indicate This Refactoring

- Related code is separated by unrelated statements
- Variables are declared far from where they're used
- Preparing to extract a function but statements are scattered
- Code is hard to follow due to statement ordering

## Mechanics

1. Identify the target position to move the statements to
2. Examine statements between source and target to see if there are any dependencies
3. If there are no dependencies, move the statements
4. Test

## Example

### Before

```java
public double calculatePrice(Order order) {
    int basePrice = order.getQuantity() * order.getItemPrice();
    int discountLevel = 0;

    // Unrelated code between variable and its use
    PricingPlan pricingPlan = retrievePricingPlan();
    double charge = order.getChargeFor(pricingPlan);

    // Now we use discountLevel
    if (basePrice > 1000) {
        discountLevel = 2;
    }

    double discount = discountLevel * 0.05 * basePrice;
    return charge - discount;
}
```

### After

```java
public double calculatePrice(Order order) {
    int basePrice = order.getQuantity() * order.getItemPrice();

    PricingPlan pricingPlan = retrievePricingPlan();
    double charge = order.getChargeFor(pricingPlan);

    // Variables now together with their use
    int discountLevel = 0;
    if (basePrice > 1000) {
        discountLevel = 2;
    }
    double discount = discountLevel * 0.05 * basePrice;

    return charge - discount;
}
```

### Preparing for Extract Function

```java
// Before - statements scattered
public void processOrder(Order order) {
    validate(order);
    double discount = calculateDiscount(order);
    log("Processing order: " + order.getId());
    double price = calculatePrice(order);
    log("Order price: " + price);
    applyDiscount(order, discount);
    save(order);
}

// After sliding - ready for extraction
public void processOrder(Order order) {
    validate(order);

    // Logging statements grouped
    log("Processing order: " + order.getId());

    // Price calculation grouped
    double discount = calculateDiscount(order);
    double price = calculatePrice(order);
    log("Order price: " + price);
    applyDiscount(order, discount);

    save(order);
}
```

## Safety Checks

Before sliding, verify:

1. The slid code doesn't reference variables that are defined between source and target
2. The slid code doesn't modify variables that are used between source and target
3. The slid code doesn't reference variables that are modified between source and target

```java
// UNSAFE to slide - dependency exists
int x = 5;
int y = x + 3;  // Can't slide this before the x declaration
print(y);

// SAFE to slide - no dependencies
int x = 5;
print("hello");  // Can slide this before x declaration
print(x);
```

## When to Use

✅ **Use Slide Statements when:**

- Preparing to extract a function
- Related code is scattered
- Variables are declared far from their use
- You want to understand code better

❌ **Avoid Slide Statements when:**

- The current ordering is intentional
- Dependencies exist between statements
- Moving would break the logic
- Current ordering reflects execution requirements

## Related Refactorings

- **Extract Function**: Often follows sliding
- **Split Variable**: May enable sliding
- **Move Statements into Function**: May follow
