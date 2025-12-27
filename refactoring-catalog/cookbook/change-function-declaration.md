# Change Function Declaration

_Also known as: Rename Function, Rename Method, Add Parameter, Remove Parameter, Change Signature_

## Intent

Change a function's name or parameters to better communicate its purpose or to change its interface.

## Code Smells That Indicate This Refactoring

- Function name doesn't clearly describe what it does
- Parameter list is confusing or too long
- Parameters are in an illogical order
- Parameter could be derived from other parameters
- Function needs additional data to work properly

## Mechanics

### Simple Mechanics (few callers, safe rename)

1. Rename the function and its parameters
2. Find all callers and update them
3. Test

### Migration Mechanics (many callers, complex change)

1. Apply **Extract Function** on the function body if necessary
2. Create a new function with the desired signature
3. Fill the new function body with a delegation to the old one (or move the logic)
4. Inline the old function body to the new function
5. Apply **Inline Function** to the old function
6. Test after each step

## Example: Rename Function

### Before

```java
public class Customer {
    public double calc(Order order) {
        double total = 0;
        for (LineItem item : order.getItems()) {
            total += item.getPrice() * item.getQuantity();
        }
        return total;
    }
}
```

### After

```java
public class Customer {
    public double calculateOrderTotal(Order order) {
        double total = 0;
        for (LineItem item : order.getItems()) {
            total += item.getPrice() * item.getQuantity();
        }
        return total;
    }
}
```

## Example: Add Parameter

### Before

```java
public class BookingService {
    public void addReservation(Customer customer) {
        // always adds as regular reservation
        this.reservations.add(new Reservation(customer, false));
    }
}
```

### After

```java
public class BookingService {
    public void addReservation(Customer customer, boolean isPriority) {
        this.reservations.add(new Reservation(customer, isPriority));
    }
}
```

## Example: Remove Parameter

### Before

```java
public class SalaryCalculator {
    public double calculateSalary(Employee employee, String unused, int year) {
        return employee.getBaseSalary() * getAnnualMultiplier(year);
    }
}
```

### After

```java
public class SalaryCalculator {
    public double calculateSalary(Employee employee, int year) {
        return employee.getBaseSalary() * getAnnualMultiplier(year);
    }
}
```

## Migration Mechanics Example

When you have many callers, use a gradual approach:

```java
// Step 1: Old method, mark deprecated
public class CircleCalculator {
    @Deprecated
    public double circum(double radius) {
        return calculateCircumference(radius);
    }

    // Step 2: New method with desired name
    public double calculateCircumference(double radius) {
        return 2 * Math.PI * radius;
    }
}

// Step 3: Update callers gradually
// Step 4: Remove deprecated method when all callers updated
```

## When to Use

✅ **Use Change Function Declaration when:**

- The function name doesn't describe what it does
- Parameters are confusing or poorly ordered
- You're adding new required behavior
- A parameter is now unnecessary
- You want to prepare for more refactoring

❌ **Avoid Change Function Declaration when:**

- The function is part of a published API
- Many callers exist and the change isn't urgent
- The current signature is well-understood by the team

## Related Refactorings

- **Extract Function**: Often precedes
- **Inline Function**: Used in migration mechanics
- **Introduce Parameter Object**: For too many parameters
- **Remove Flag Argument**: Specific case of parameter removal
