# Move Field

## Intent

Move a field from one class to another that uses it more or provides better context for it.

## Code Smells That Indicate This Refactoring

- Field is used more in another class
- Field is passed as a parameter to many methods of another class
- Field seems out of place conceptually
- When changing the field, you always need to update another class

## Mechanics

1. Ensure the source field is encapsulated
2. Create a field (and accessors) in the target class
3. Run static checks
4. Ensure there is a reference from the source object to the target object
5. Update all users of the source field to use the target field
6. Test
7. Remove the source field

## Example

### Before

```java
public class Customer {
    private String name;
    private double discountRate;
    private CustomerContract contract;

    public double getDiscountRate() {
        return discountRate;
    }

    public void becomePreferred() {
        discountRate += 0.03;
        // notify contract of change...
    }

    public double applyDiscount(double amount) {
        return amount - (amount * discountRate);
    }
}
```

### After

```java
public class CustomerContract {
    private LocalDate startDate;
    private double discountRate;

    public CustomerContract(LocalDate startDate, double discountRate) {
        this.startDate = startDate;
        this.discountRate = discountRate;
    }

    public double getDiscountRate() {
        return discountRate;
    }

    public void setDiscountRate(double discountRate) {
        this.discountRate = discountRate;
    }
}

public class Customer {
    private String name;
    private CustomerContract contract;

    public double getDiscountRate() {
        return contract.getDiscountRate();
    }

    public void becomePreferred() {
        contract.setDiscountRate(contract.getDiscountRate() + 0.03);
    }

    public double applyDiscount(double amount) {
        return amount - (amount * getDiscountRate());
    }
}
```

### Moving Between Related Objects

```java
// Before - account holds interest rate
public class Account {
    private AccountType type;
    private double interestRate;

    public double getInterestRate() {
        return interestRate;
    }
}

// After - interest rate belongs on account type
public class AccountType {
    private String name;
    private double interestRate;

    public double getInterestRate() {
        return interestRate;
    }
}

public class Account {
    private AccountType type;

    public double getInterestRate() {
        return type.getInterestRate();
    }
}
```

## When to Use

✅ **Use Move Field when:**

- Field is used more heavily by another class
- Field is conceptually part of another abstraction
- Moving would reduce coupling
- Field and related behavior should move together

❌ **Avoid Move Field when:**

- It would create circular dependencies
- The field is integral to the current class's identity
- Moving would increase coupling overall
- The field is accessed through inheritance

## Related Refactorings

- **Move Function**: Often done with field movement
- **Extract Class**: May precede to create the target class
- **Encapsulate Variable**: Must be done first if field is public
