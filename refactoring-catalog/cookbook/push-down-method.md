# Push Down Method

## Intent

Move a method from a superclass to the subclass(es) that use it.

## Code Smells That Indicate This Refactoring

- Method only relevant to some subclasses
- Method in superclass throws "not implemented" for some subclasses
- Method accesses subclass-specific features
- Method doesn't make sense for all subclass types

## Mechanics

1. Copy the method to each subclass that needs it
2. Remove the method from the superclass
3. Test
4. Remove from subclasses that don't need it
5. Test

## Example

### Before

```java
public abstract class Employee {
    protected String name;
    protected double baseSalary;

    public double getQuota() {
        // Only makes sense for Salesperson
        throw new UnsupportedOperationException("Only salespeople have quotas");
    }
}

public class Engineer extends Employee {
    // Doesn't need getQuota()
}

public class Salesperson extends Employee {
    private double quota;

    @Override
    public double getQuota() {
        return quota;
    }
}
```

### After

```java
public abstract class Employee {
    protected String name;
    protected double baseSalary;
    // No getQuota() method
}

public class Engineer extends Employee {
    // No quota concept
}

public class Salesperson extends Employee {
    private double quota;

    public double getQuota() {
        return quota;
    }
}
```

### Multiple Subclasses Need It

```java
// Before
public abstract class Bird {
    public String getPlumage() {
        return "average";
    }
}

// After - only flying birds have typical plumage
public abstract class Bird {
    // No plumage method - not all birds have plumage relevant to this app
}

public abstract class FlyingBird extends Bird {
    public String getPlumage() {
        return "average";
    }
}

public class Penguin extends Bird {
    // No plumage method needed
}

public class Sparrow extends FlyingBird {
    // Inherits getPlumage()
}
```

### Interface Alternative

```java
// When behavior is optional, consider interface
public interface QuotaHolder {
    double getQuota();
    void setQuota(double quota);
}

public abstract class Employee {
    protected String name;
}

public class Engineer extends Employee {
    // No quota
}

public class Salesperson extends Employee implements QuotaHolder {
    private double quota;

    @Override
    public double getQuota() {
        return quota;
    }

    @Override
    public void setQuota(double quota) {
        this.quota = quota;
    }
}

// Client code can check for capability
if (employee instanceof QuotaHolder) {
    ((QuotaHolder) employee).getQuota();
}
```

## When to Use

✅ **Use Push Down Method when:**

- Method only makes sense for some subclasses
- Superclass has to throw exceptions for some subclasses
- Method accesses subclass-specific state
- Different subclasses would implement it differently

❌ **Avoid Push Down Method when:**

- Most subclasses need the method
- The method is part of the public API
- Pushing down would create duplication
- The method is abstract and should be implemented by all

## Related Refactorings

- **Push Down Field**: Often done together
- **Pull Up Method**: The inverse operation
- **Replace Conditional with Polymorphism**: Alternative approach
- **Extract Superclass**: To reorganize hierarchy
