# Move Function

_Also known as: Move Method_

## Intent

Move a function to the module or class that uses it most or that provides most of its dependencies.

## Code Smells That Indicate This Refactoring

- Function uses more features of another class than its own
- Function is in a different module than the data it operates on
- Related functions are scattered across classes
- Adding a new feature requires editing multiple classes

## Mechanics

1. Examine all elements used by the function in its current context
2. Check if the function is polymorphic
3. Copy the function to the target context and name it appropriately
4. Adjust the function to fit the new context
5. Make the old function delegate to the new one (or remove it)
6. Test

## Example

### Before

```java
public class Account {
    private AccountType type;
    private int daysOverdrawn;

    public double getBankCharge() {
        double result = 4.5;
        if (daysOverdrawn > 0) {
            result += getOverdraftCharge();
        }
        return result;
    }

    private double getOverdraftCharge() {
        if (type.isPremium()) {
            double baseCharge = 10;
            if (daysOverdrawn <= 7) {
                return baseCharge;
            } else {
                return baseCharge + (daysOverdrawn - 7) * 0.85;
            }
        } else {
            return daysOverdrawn * 1.75;
        }
    }
}
```

### After

```java
public class AccountType {
    private boolean premium;

    public boolean isPremium() {
        return premium;
    }

    public double getOverdraftCharge(int daysOverdrawn) {
        if (isPremium()) {
            double baseCharge = 10;
            if (daysOverdrawn <= 7) {
                return baseCharge;
            } else {
                return baseCharge + (daysOverdrawn - 7) * 0.85;
            }
        } else {
            return daysOverdrawn * 1.75;
        }
    }
}

public class Account {
    private AccountType type;
    private int daysOverdrawn;

    public double getBankCharge() {
        double result = 4.5;
        if (daysOverdrawn > 0) {
            result += type.getOverdraftCharge(daysOverdrawn);
        }
        return result;
    }
}
```

### Moving to Utility Class

```java
// Before - utility method in wrong class
public class Order {
    public String formatAddress(Address address) {
        return address.getStreet() + "\n" +
               address.getCity() + ", " + address.getState() + " " + address.getZip();
    }
}

// After - method belongs with its data
public class Address {
    private String street;
    private String city;
    private String state;
    private String zip;

    public String format() {
        return street + "\n" + city + ", " + state + " " + zip;
    }
}
```

## When to Use

✅ **Use Move Function when:**

- Function references features of another class more than its own
- Function's name suggests it belongs elsewhere
- Similar functions are already in the target class
- Moving would reduce parameter passing

❌ **Avoid Move Function when:**

- The function is polymorphic and overridden in subclasses
- Moving would create circular dependencies
- The current location is correct for encapsulation
- The function is a legitimate coordinator/orchestrator

## Related Refactorings

- **Move Field**: Often accompanies function movement
- **Extract Function**: May precede move
- **Inline Function**: May be done instead
