# Decompose Conditional

## Intent

Extract the condition and each branch of an if-else statement into separate methods to make the logic clearer.

## Code Smells That Indicate This Refactoring

- Complex conditional with long condition expression
- If/else branches with substantial logic
- Condition logic obscuring what the code decides
- Duplicate code in condition checks

## Mechanics

1. Apply **Extract Function** on the condition
2. Apply **Extract Function** on the then-leg
3. Apply **Extract Function** on the else-leg (if present)

## Example

### Before

```java
public class Rental {
    private LocalDate startDate;
    private LocalDate endDate;
    private double baseRate;
    private double winterRate;
    private double winterServiceCharge;

    public double calculateCharge() {
        double charge;
        if (startDate.getMonthValue() >= 11 || startDate.getMonthValue() <= 2) {
            charge = ChronoUnit.DAYS.between(startDate, endDate) * winterRate + winterServiceCharge;
        } else {
            charge = ChronoUnit.DAYS.between(startDate, endDate) * baseRate;
        }
        return charge;
    }
}
```

### After

```java
public class Rental {
    private LocalDate startDate;
    private LocalDate endDate;
    private double baseRate;
    private double winterRate;
    private double winterServiceCharge;

    public double calculateCharge() {
        if (isWinter()) {
            return winterCharge();
        } else {
            return regularCharge();
        }
    }

    private boolean isWinter() {
        int month = startDate.getMonthValue();
        return month >= 11 || month <= 2;
    }

    private double winterCharge() {
        return rentalDays() * winterRate + winterServiceCharge;
    }

    private double regularCharge() {
        return rentalDays() * baseRate;
    }

    private long rentalDays() {
        return ChronoUnit.DAYS.between(startDate, endDate);
    }
}
```

### With Ternary (Simple Cases)

```java
// Before
public double calculateCharge() {
    if (isWinter()) {
        return winterCharge();
    } else {
        return regularCharge();
    }
}

// After (when branches are simple)
public double calculateCharge() {
    return isWinter() ? winterCharge() : regularCharge();
}
```

## When to Use

✅ **Use Decompose Conditional when:**

- The condition expression is complex
- The then/else branches have substantial logic
- You want to name the condition for clarity
- You see similar conditions elsewhere

❌ **Avoid Decompose Conditional when:**

- The conditional is already simple and clear
- Extracting would add more complexity
- The branches are single, simple statements
- The meaning is obvious from the code

## Related Refactorings

- **Extract Function**: The core technique used
- **Consolidate Conditional Expression**: When multiple conditions have same result
- **Replace Conditional with Polymorphism**: When the condition is type-based
- **Replace Nested Conditional with Guard Clauses**: For deep nesting
