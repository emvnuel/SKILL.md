# Consolidate Conditional Expression

## Intent

Combine multiple conditional tests that have the same result into a single conditional expression.

## Code Smells That Indicate This Refactoring

- Sequential if statements returning the same value
- Multiple conditions with identical outcomes
- Logic spread across multiple checks that could be unified
- Related guard clauses that could be combined

## Mechanics

1. Ensure none of the conditionals have side effects (if they do, use **Separate Query from Modifier** first)
2. Combine two of the conditional statements using logical operators
3. Test
4. Repeat combining until all are merged
5. Consider **Extract Function** on the combined condition

## Example

### Before

```java
public class Employee {
    private int seniority;
    private int monthsDisabled;
    private boolean isPartTime;

    public double calculateDisabilityAmount() {
        if (seniority < 2) {
            return 0;
        }
        if (monthsDisabled > 12) {
            return 0;
        }
        if (isPartTime) {
            return 0;
        }
        // compute the disability amount
        return calculateFullDisabilityAmount();
    }
}
```

### After

```java
public class Employee {
    private int seniority;
    private int monthsDisabled;
    private boolean isPartTime;

    public double calculateDisabilityAmount() {
        if (isNotEligibleForDisability()) {
            return 0;
        }
        return calculateFullDisabilityAmount();
    }

    private boolean isNotEligibleForDisability() {
        return seniority < 2
            || monthsDisabled > 12
            || isPartTime;
    }
}
```

### Using OR

```java
// Before
public boolean shouldNotify(User user) {
    if (user.isAdmin()) {
        return true;
    }
    if (user.isOwner()) {
        return true;
    }
    if (user.hasExplicitPermission()) {
        return true;
    }
    return false;
}

// After
public boolean shouldNotify(User user) {
    return user.isAdmin()
        || user.isOwner()
        || user.hasExplicitPermission();
}
```

### Using AND

```java
// Before
public boolean isValidOrder(Order order) {
    if (!order.hasItems()) {
        return false;
    }
    if (!order.hasValidPayment()) {
        return false;
    }
    if (!order.hasShippingAddress()) {
        return false;
    }
    return true;
}

// After
public boolean isValidOrder(Order order) {
    return order.hasItems()
        && order.hasValidPayment()
        && order.hasShippingAddress();
}
```

## When to Use

✅ **Use Consolidate Conditional Expression when:**

- Multiple conditionals have the same result
- The checks represent one logical concept
- Combining makes the intent clearer
- You want to extract the combined condition

❌ **Avoid Consolidate Conditional Expression when:**

- The conditions are truly independent
- They might need to diverge in the future
- Combining would make the logic less clear
- The conditions have side effects

## Related Refactorings

- **Extract Function**: Often applied after consolidation
- **Decompose Conditional**: Inverse—splits complex conditions
- **Replace Conditional with Polymorphism**: For type-based conditions
