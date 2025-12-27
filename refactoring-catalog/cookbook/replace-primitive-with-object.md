# Replace Primitive with Object

_Also known as: Replace Data Value with Object, Replace Type Code with Class_

## Intent

Replace a primitive value that has behavior or meaning beyond being a simple value with an object that encapsulates that behavior.

## Code Smells That Indicate This Refactoring

- Primitive value (string, number) represents a domain concept
- Validation logic scattered wherever the primitive is used
- Formatting logic duplicated across the codebase
- Multiple primitives always travel together (could become one object)
- Primitive value has special operations or comparisons

## Mechanics

1. Apply **Encapsulate Variable** if it isn't already
2. Create a simple value class for the data value, with a constructor and getter
3. Run static checks
4. Change the setter to create a new instance of the class
5. Change the getter to return the result of invoking the getter of the new class
6. Test
7. Consider using **Rename Variable** on the original accessors to better reflect what they do
8. Add new behavior to the class as needed

## Example

### Before

```java
public class Order {
    private String customer;

    public Order(String customer) {
        this.customer = customer;
    }

    public String getCustomer() {
        return customer;
    }

    public void setCustomer(String customer) {
        this.customer = customer;
    }
}

// Usage
Order order = new Order("Acme Corp");
String customerName = order.getCustomer();
```

### After

```java
public class Customer {
    private final String name;

    public Customer(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Customer customer = (Customer) o;
        return Objects.equals(name, customer.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);
    }
}

public class Order {
    private Customer customer;

    public Order(String customerName) {
        this.customer = new Customer(customerName);
    }

    public String getCustomerName() {
        return customer.getName();
    }

    public Customer getCustomer() {
        return customer;
    }

    public void setCustomer(String customerName) {
        this.customer = new Customer(customerName);
    }
}
```

### Extended Example with Behavior

```java
public class PhoneNumber {
    private final String number;

    public PhoneNumber(String number) {
        if (!isValid(number)) {
            throw new IllegalArgumentException("Invalid phone number: " + number);
        }
        this.number = normalize(number);
    }

    private static boolean isValid(String number) {
        return number != null && number.matches("\\+?[0-9\\s\\-()]+");
    }

    private static String normalize(String number) {
        return number.replaceAll("[\\s\\-()]", "");
    }

    public String getAreaCode() {
        return number.substring(0, 3);
    }

    public String format() {
        return String.format("(%s) %s-%s",
            number.substring(0, 3),
            number.substring(3, 6),
            number.substring(6));
    }

    public String getNumber() {
        return number;
    }
}
```

## When to Use

✅ **Use Replace Primitive with Object when:**

- The primitive has validation rules
- The primitive needs formatting logic
- The primitive has associated behavior
- Multiple primitives always travel together
- You're doing complex comparisons on the primitive

❌ **Avoid Replace Primitive with Object when:**

- The value is truly just data with no behavior
- The overhead isn't justified by the complexity
- The primitive is only used in one place
- Adding the class would over-engineer the solution

## Related Refactorings

- **Encapsulate Variable**: Often applied first
- **Change Reference to Value**: When the object should be immutable
- **Introduce Parameter Object**: For grouping related primitives
- **Replace Type Code with Subclasses**: For type codes with behavior
