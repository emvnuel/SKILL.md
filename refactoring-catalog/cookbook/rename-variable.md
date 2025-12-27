# Rename Variable

## Intent

Change a variable name to better communicate its purpose.

## Code Smells That Indicate This Refactoring

- Variable name is cryptic or abbreviated
- Name doesn't match what the variable actually represents
- Name uses outdated terminology
- Name is too generic (data, value, temp, result)
- Single-letter names in non-trivial scopes

## Mechanics

1. If the variable is widely used, consider **Encapsulate Variable** first
2. Find all references to the variable and rename them
3. Test

## Example

### Before

```java
public class ShoppingCart {
    public double calculate(List<Item> i) {
        double t = 0;
        for (Item x : i) {
            double p = x.getPrice();
            int q = x.getQuantity();
            t += p * q;
        }
        double d = t > 100 ? 0.1 : 0;
        return t - (t * d);
    }
}
```

### After

```java
public class ShoppingCart {
    public double calculateTotal(List<Item> items) {
        double subtotal = 0;
        for (Item item : items) {
            double itemPrice = item.getPrice();
            int quantity = item.getQuantity();
            subtotal += itemPrice * quantity;
        }
        double discountRate = subtotal > 100 ? 0.1 : 0;
        return subtotal - (subtotal * discountRate);
    }
}
```

### Field Renaming

```java
// Before
public class Person {
    private String nm;
    private int yob;

    public String getNm() { return nm; }
    public int getYob() { return yob; }
}

// After
public class Person {
    private String name;
    private int yearOfBirth;

    public String getName() { return name; }
    public int getYearOfBirth() { return yearOfBirth; }
}
```

## Naming Guidelines

| Bad              | Better                            | Reason           |
| ---------------- | --------------------------------- | ---------------- |
| `d`, `n`, `x`    | `distance`, `name`, `xCoordinate` | Descriptive      |
| `data`, `value`  | `customerData`, `totalValue`      | More specific    |
| `tmp`, `temp`    | `temporaryFile`, `tempPassword`   | Purpose clear    |
| `flag`, `status` | `isEnabled`, `orderStatus`        | Explains meaning |
| `list`, `arr`    | `customers`, `orderItems`         | Content-focused  |

## When to Use

✅ **Use Rename Variable when:**

- You had to look at the code to understand what the variable holds
- The variable name is misleading
- Domain terminology has changed
- You're cleaning up legacy code
- Single-letter names exist outside of tiny scopes (loops)

❌ **Avoid excessive renaming when:**

- The name is already clear
- It's a well-understood convention (i for loop index)
- The variable scope is tiny and obvious
- Renaming would break external APIs

## Related Refactorings

- **Encapsulate Variable**: When the variable is widely used
- **Rename Field**: For class fields specifically
- **Change Function Declaration**: For renaming parameters
