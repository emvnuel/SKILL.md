# Push Down Field

## Intent

Move a field from a superclass to the subclass(es) that use it.

## Code Smells That Indicate This Refactoring

- Field only used by some subclasses
- Field doesn't make sense for all subclasses
- Field always null or default for some subclasses

## Mechanics

1. Declare the field in each subclass that needs it
2. Remove the field from the superclass
3. Test

## Example

### Before

```java
public abstract class Employee {
    protected String name;
    protected double baseSalary;
    protected double commission; // Only salespeople have commission
}

public class Engineer extends Employee {
    // commission is never used
}

public class Salesperson extends Employee {
    public double calculateEarnings() {
        return baseSalary + commission;
    }
}
```

### After

```java
public abstract class Employee {
    protected String name;
    protected double baseSalary;
}

public class Engineer extends Employee {
    // No commission field
}

public class Salesperson extends Employee {
    private double commission;

    public double getCommission() {
        return commission;
    }

    public void setCommission(double commission) {
        this.commission = commission;
    }

    public double calculateEarnings() {
        return baseSalary + commission;
    }
}
```

### To Multiple Subclasses

```java
// Before
public abstract class Vehicle {
    protected String make;
    protected String model;
    protected double fuelCapacity; // Not all vehicles use fuel
}

// After
public abstract class Vehicle {
    protected String make;
    protected String model;
}

public abstract class FuelVehicle extends Vehicle {
    protected double fuelCapacity;

    public double getFuelCapacity() {
        return fuelCapacity;
    }
}

public class Car extends FuelVehicle {
    // Has fuelCapacity
}

public class ElectricCar extends Vehicle {
    private double batteryCapacity;
    // No fuel capacity
}
```

## When to Use

✅ **Use Push Down Field when:**

- Field only makes sense for some subclasses
- Field is always null/zero for some subclasses
- Field is tightly coupled to subclass-specific behavior
- Superclass shouldn't know about subclass details

❌ **Avoid Push Down Field when:**

- Most subclasses need the field
- The field is part of the common base behavior
- Pushing down would create duplication
- The field is used polymorphically

## Related Refactorings

- **Push Down Method**: Often done together
- **Pull Up Field**: The inverse operation
- **Extract Subclass**: May be triggered by this
