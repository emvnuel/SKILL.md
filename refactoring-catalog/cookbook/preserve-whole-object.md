# Preserve Whole Object

## Intent

Replace passing multiple values from an object with passing the whole object.

## Code Smells That Indicate This Refactoring

- Extracting several values from an object to pass as separate arguments
- Long parameter lists that come from the same source
- Multiple methods receiving the same group of parameters
- Changes to the object require updating many call sites

## Mechanics

1. Create an empty method with the desired parameters
2. Fill in the body with a call to the old method, mapping from new parameters to old
3. Run static checks
4. Replace each old call with the new, testing after each change
5. Once all calls use the new method, use **Inline Function** to inline the old into the new
6. Apply **Change Function Declaration** to rename if needed

## Example

### Before

```java
public class Room {
    private TemperatureRange daysTempRange;

    public boolean isWithinPlan(HeatingPlan plan) {
        int low = daysTempRange.getLow();
        int high = daysTempRange.getHigh();
        return plan.withinRange(low, high);
    }
}

public class HeatingPlan {
    private int temperatureFloor;
    private int temperatureCeiling;

    public boolean withinRange(int low, int high) {
        return low >= temperatureFloor && high <= temperatureCeiling;
    }
}
```

### After

```java
public class Room {
    private TemperatureRange daysTempRange;

    public boolean isWithinPlan(HeatingPlan plan) {
        return plan.withinRange(daysTempRange);
    }
}

public class HeatingPlan {
    private int temperatureFloor;
    private int temperatureCeiling;

    public boolean withinRange(TemperatureRange range) {
        return range.getLow() >= temperatureFloor
            && range.getHigh() <= temperatureCeiling;
    }
}
```

### Order Processing Example

```java
// Before
public class OrderProcessor {
    public void processOrder(String customerId, String customerName,
                            String email, String shippingAddress) {
        // processing logic
    }
}

// Called as:
processor.processOrder(
    customer.getId(),
    customer.getName(),
    customer.getEmail(),
    customer.getShippingAddress()
);

// After
public class OrderProcessor {
    public void processOrder(Customer customer) {
        // processing logic using customer.getId(), etc.
    }
}

// Called as:
processor.processOrder(customer);
```

### Creating Parameter Object If Needed

```java
// If there's no suitable object, create one first
public record TemperatureRange(int low, int high) {
    public boolean contains(int temperature) {
        return temperature >= low && temperature <= high;
    }
}
```

## When to Use

✅ **Use Preserve Whole Object when:**

- You're extracting multiple values from the same object
- The object provides meaningful context
- Multiple methods use the same group of values
- Adding new requirements would need more values from the object

❌ **Avoid Preserve Whole Object when:**

- The caller shouldn't depend on the whole object
- The function should be pure with primitive inputs
- Passing the object would create unwanted coupling
- Only one or two values are needed

## Coupling Consideration

This refactoring may introduce a dependency on the whole object type, which could be undesirable:

```java
// Might create unwanted coupling
public void sendNotification(Customer customer) {
    // Now NotificationService depends on Customer class
}

// Alternative: keep primitives or use interface
public void sendNotification(String email, String name) {
    // NotificationService doesn't know about Customer
}
```

## Related Refactorings

- **Introduce Parameter Object**: When you need to create the object first
- **Replace Parameter with Query**: Alternative approach
- **Move Function**: May follow if the function should move to the passed object
