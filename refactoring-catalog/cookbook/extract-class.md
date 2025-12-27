# Extract Class

## Intent

Split a class that does too much into two classes, each with a clear responsibility.

## Code Smells That Indicate This Refactoring

- Class has too many responsibilities
- Groups of fields that go together
- Methods that operate on a subset of fields
- Class is hard to understand or test
- Changing one feature affects unrelated code

## Mechanics

1. Decide how to split responsibilities
2. Create a new class for a separated responsibility
3. Hold an instance of the new class in the old class
4. Use **Move Field** for each field that belongs in the new class
5. Use **Move Function** for each method in the new class
6. Review and reduce the interfaces of both classes
7. Decide whether to expose the new class

## Example

### Before

```java
public class Person {
    private String name;
    private String officeAreaCode;
    private String officeNumber;
    private String homeAreaCode;
    private String homeNumber;

    public String getName() { return name; }

    public String getOfficeAreaCode() { return officeAreaCode; }
    public void setOfficeAreaCode(String code) { this.officeAreaCode = code; }

    public String getOfficeNumber() { return officeNumber; }
    public void setOfficeNumber(String number) { this.officeNumber = number; }

    public String getHomeAreaCode() { return homeAreaCode; }
    public String getHomeNumber() { return homeNumber; }

    public String getOfficeTelephoneNumber() {
        return "(" + officeAreaCode + ") " + officeNumber;
    }

    public String getHomeTelephoneNumber() {
        return "(" + homeAreaCode + ") " + homeNumber;
    }
}
```

### After

```java
public class TelephoneNumber {
    private String areaCode;
    private String number;

    public TelephoneNumber(String areaCode, String number) {
        this.areaCode = areaCode;
        this.number = number;
    }

    public String getAreaCode() { return areaCode; }
    public void setAreaCode(String areaCode) { this.areaCode = areaCode; }

    public String getNumber() { return number; }
    public void setNumber(String number) { this.number = number; }

    public String getFormatted() {
        return "(" + areaCode + ") " + number;
    }

    @Override
    public String toString() {
        return getFormatted();
    }
}

public class Person {
    private String name;
    private TelephoneNumber officePhone;
    private TelephoneNumber homePhone;

    public Person(String name) {
        this.name = name;
        this.officePhone = new TelephoneNumber("", "");
        this.homePhone = new TelephoneNumber("", "");
    }

    public String getName() { return name; }

    public TelephoneNumber getOfficePhone() { return officePhone; }
    public TelephoneNumber getHomePhone() { return homePhone; }

    public String getOfficeTelephoneNumber() {
        return officePhone.getFormatted();
    }

    public String getHomeTelephoneNumber() {
        return homePhone.getFormatted();
    }
}
```

### Service Extraction

```java
// Before - Order does too much
public class Order {
    private Customer customer;
    private List<LineItem> items;

    public double getTotal() { /* ... */ }

    public void sendConfirmationEmail() { /* ... */ }
    public void processPayment(PaymentDetails details) { /* ... */ }
    public void reserveInventory() { /* ... */ }
    public void scheduleShipping() { /* ... */ }
}

// After - separate concerns
public class Order {
    private Customer customer;
    private List<LineItem> items;

    public double getTotal() { /* ... */ }
    public Customer getCustomer() { return customer; }
    public List<LineItem> getItems() { return items; }
}

public class OrderFulfillmentService {
    public void fulfill(Order order, PaymentDetails payment) {
        processPayment(order, payment);
        reserveInventory(order);
        scheduleShipping(order);
        sendConfirmationEmail(order);
    }

    private void processPayment(Order order, PaymentDetails payment) { /* ... */ }
    private void reserveInventory(Order order) { /* ... */ }
    private void scheduleShipping(Order order) { /* ... */ }
    private void sendConfirmationEmail(Order order) { /* ... */ }
}
```

## When to Use

✅ **Use Extract Class when:**

- Class has grown too large
- Groups of fields/methods are cohesive
- Class has multiple responsibilities
- You want smaller, focused classes

❌ **Avoid Extract Class when:**

- Class is already focused
- Splitting would create artificial boundaries
- The extracted class would have no clear responsibility
- It would over-complicate the design

## Related Refactorings

- **Inline Class**: The inverse operation
- **Move Field**: Used during extraction
- **Move Function**: Used during extraction
- **Extract Superclass**: Alternative for type-based extraction
